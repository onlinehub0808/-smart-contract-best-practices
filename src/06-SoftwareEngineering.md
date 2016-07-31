
## Software Engineering Techniques

As we discussed in the [General Philosophy](#general-philosophy) section, it is not enough to protect yourself against the known attacks. Since the cost of failure on a blockchain can be very high, you must also adapt the way you write software, to account for that risk.

The approach we advocate is to "prepare for failure". It is impossible to know in advance whether your code is secure. However, you can architect your contracts in a way that allows them to fail gracefully, and with minimal damage. This section presents a variety of techniques that will help you prepare for failure.

Note: There's always a risk when you add a new component to your system. A badly designed fail-safe could itself become a vulnerability - as can the interaction between a number of well designed fail-safes. Be thoughtful about each technique you use in your contracts, and consider carefully how they work together to create a robust system.

### Upgrading Broken Contracts

Code will need to be changed if errors are discovered or if improvements need to be made. It is no good to discover a bug, but have no way to deal with it.

Designing an effective upgrade system for smart contracts is an area of active research, and we won't be able to cover all of the complications in this document. However, there are two basic approaches that are most commonly used. The simpler of the two is to have a registry contract that holds the address of the latest version of the contract. A more seamless approach for contract users is to have a contract that forwards calls and data onto the latest version of the contract.

Whatever the technique, it's important to have modularization and good separation between components, so that code changes do not break functionality, orphan data, or require substantial costs to port. In particular, it is usually beneficial to separate complex logic from your data storage, so that you do not have to recreate all of the data in order to change the functionality.

It's also critical to have a secure way for parties to decide to upgrade the code. Depending on your contract, code changes may need to be approved by a single trusted party, a group of members, or a vote of the full set of stakeholders. If this process can take some time, you will want to consider if there are other ways to react more quickly in case of an attack, such as an [emergency stop or circuit-breaker](https://github.com/ConsenSys/smart-contract-best-practices/#circuit-breakers-pause-contract-functionality).

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

There are two main disadvantages to this approach:

1. Users must always look up the current address, and anyone who fails to do so risks using an old version of the contract
2. You will need to think carefully about how to deal with the contract data, when you replace the contract

The alternate approach is to have a contract forward calls and data to the latest version of the contract:

**Example 2: [Use a `DELEGATECALL`](http://ethereum.stackexchange.com/questions/2404/upgradeable-contracts) to forward data and calls**

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

This approach avoids the previous problems, but has problems of its own. You must be extremely careful with how you store data in this contract. If your new contract has a different storage layout than the first, your data may end up corrupted. Additionally, this simple version of the pattern cannot return values from functions, only forward them, which limits its applicability. ([More complex implementations](https://github.com/ownage-ltd/ether-router) attempt to solve this with in-line assembly code and a registry of return sizes.)

Regardless of your approach, it is important to have some way to upgrade your contracts, or they will become unusable when the inevitable bugs are discovered in them.

### Circuit Breakers (Pause contract functionality)

Circuit breakers stop execution if certain conditions are met, and can be useful when new errors are discovered. For example, most actions may be suspended in a contract if a bug is discovered, and the only action now active is a withdrawal. You can either give certain trusted parties the ability to trigger the circuit breaker, or else have programmatic rules that automatically trigger the certain breaker when certain conditions are met.

Example:

```
bool private stopped = false;
address private owner;

modifier isAdmin() {
    if(msg.sender != owner) {
        throw;
    }
    _
}

function toggleContractActive() isAdmin public
{
    // You can add an additional modifier that restricts stopping a contract to be based on another action, such as a vote of users
    stopped = !stopped;
}

modifier stopInEmergency { if (!stopped) _ }
modifier onlyInEmergency { if (stopped) _ }

function deposit() stopInEmergency public
{
    // some code
}

function withdraw() onlyInEmergency public
{
    // some code
}
```

### Speed Bumps (Delay contract actions)

Speed bumps slow down actions, so that if malicious actions occur, there is time to recover. For example, [The DAO](https://github.com/slockit/DAO/) required 27 days between a successful request to split the DAO and the ability to do so. This ensured the funds were kept within the contract, increasing the likelihood of recovery. In the case of the DAO, there was no effective action that could be taken during the time given by the speed bump, but in combination with our other techniques, they can be quite effective.

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

Rate limiting halts or requires approval for substantial changes. For example, a depositor may only be allowed to withdraw a certain amount or percentage of total deposits over a certain time period (e.g., max 100 ether over 1 day) - additional withdrawals in that time period may fail or require some sort of special approval. Or the rate limit could be at the contract level, with only a certain amount of tokens issued by the contract over a time period.

[Example](https://gist.github.com/PeterBorah/110c331dca7d23236f80e69c83a9d58c#file-circuitbreaker-sol)

### Assert Guards

An assert guard triggers when an assertion fails - such as an invariant property changing. For example, the token to ether issuance ratio, in a token issuance contract, may be fixed. You can verify that this is the case at all times with an assertion. Assert guards should often be combined with other techniques, such as pausing the contract and allowing upgrades. (Otherwise you may end up stuck, with an assertion that is always failing.)

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
    if (block.number > SOME_BLOCK_NUMBER) {
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
