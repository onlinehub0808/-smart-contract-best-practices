이 페이지에서는 스마트 컨트랙트를 작성할 때 일반적으로 꼭 따라야 하는 여러가지 솔리디티 패턴들을 살펴보도록 하겠습니다.

## 프로토콜에 따른 추천사항들

### <!-- -->

다음 추천사항들은 이더리움 상의 모든 컨트랙트 시스템에 적용되는 사항들입니다.

## 외부 호출(External Calls)

### 외부 호출을 만들 때 경고(caution) 사용하기

신뢰할 수 없는 컨트랙트를 호출하면 예상치 못한 여러가지 위험과 에러를 마주할 수 있습니다. 외부 호출은 해당 컨트랙트 내부의 혹은 의존성을 가진 다른 연결된 컨트랙트 내부에 존재하는 악의적인 코드를 실행할 수 있습니다. 따라서 모든 외부 호출은 반드시 잠재적인 보안 위협으로 간주해야 합니다. 외부호출을 제거하는 것이 불가능하거나 어려운 상황이라면 본 섹션의 나머지 항목에서 소개하는 추천 사항들을 통해 위험요소를 최소화하기 바랍니다.

### 신뢰할 수 없는 컨트랙트 표시하기

외부 컨트랙트와 상호작용할 때, 변수와 메소드 그리고 컨트랙트 인터페이스의 이름을 잠재적으로 안전하지 않은 상호작용을 하는 것임을 한 눈에 알아볼 수 있도록 명명합니다. 이 규칙을 외부 컨트랙트를 호출하는 함수들에게 적용합니다.

```sol
// 나쁜 케이스
Bank.withdraw(100); // 신뢰할 수 있는지 아닌지 불분명합니다

function makeWithdrawal(uint amount) { // 이 함수가 잠재적으로 위험한지 불분명합니다
    Bank.withdraw(amount);
}

// 좋은 케이스
UntrustedBank.withdraw(100); // 신뢰할 수 없는 외부 호출
TrustedBank.withdraw(100); // 외부 호출이지만 XYZ Corp가 운영하는 신뢰가능한 컨트랙트에 대한 호출

function makeUntrustedWithdrawal(uint amount) {
    UntrustedBank.withdraw(amount);
}
```


### 외부 호출 이후에 상태 변경하지 않기

`someAddress.call()`과 같은 형태의 호출(*raw call*)이나 `ExternalContract.someMethod()`와 같은 형태의 호출(*contract call*)을 사용할 때, 악의적인 코드가 실행될 것이라고 가정해야 합니다. 심지어 `ExternalContract`가 악의적이지 않더라도 *그 컨트랙트* 가 실행하는 다른 어떤 컨트랙트에 의해 악의적인 코드가 실행될 수도 있습니다.

특히, 악의적인 코드가 제어 흐름을 하이재킹해 경쟁 상태(Race Condition)로 이끄는 공격이 있습니다. (이 문제에 대한 토론을 더 자세하게 보기 위해서는 [Race Conditions](./known_attacks#race-conditions) 항목으로 이동하세요).

만일 신뢰할 수 없는 외부 컨트랙트에 대한 호출을 만들고 있다면 호출 이후에는 상태를 변경하지 마세요. 이 패턴은 [checks-effects-interactions pattern](http://solidity.readthedocs.io/en/develop/security-considerations.html?highlight=check%20effects#use-the-checks-effects-interactions-pattern)로도 알려져 있습니다.


### `send()`와 `transfer()` 그리고 `call.value()()`간의 트레이드오프에 대해 파악하기

이더를 전송할 때, `someAddress.send()`, `someAddress.transfer()`, 그리고 `someAddress.call.value()()` 항목 사이의 상대적인 장단점 대해 알고 있어야 합니다.

- `someAddress.send()`와 `someAddress.transfer()` 방식을 사용하는 것은 [reentrancy](./known_attacks#reentrancy) 공격에 대해 상대적으로 안전한 것으로 간주되고 있습니다.
    위 메소드들이 코드 실행을 트리거할 때, 호출된 컨트랙트에는 이벤트를 기록하기에 충분한 양인 2,300 가스만 주어집니다.
- `x.transfer(y)` 표현은 `require(x.send(y));`과 같습니다. 이 것은 송금이 실패했을 때 자동으로 트랜잭션을 무효화(revert) 시킵니다.
- `someAddress.call.value(y)()` 표현은 적시된 이더리움을 송금하고 코드 실행을 트리거합니다.
    이때는 실행되는 코드에 가용 가스 전부가 주어집니다. 따라서 이러한 방식의 전송은 재진입(reentrancy) 공격에 대해 *안전하지 않습니다*.


`send()` 혹은 `transfer()` 메소드를 사용하면 재진입(reentrancy) 공격을 예방할 수 있지만, 2300 가스 이상을 사용하는 폴백 함수(fallback function)를 가진 컨트랙트들과는 호환되지 않는 단점이 있습니다. 혹은 `someAddress.call.value(ethAmount).gas(gasAmount)()` 형태의 표현을 통해 지정된 양의 가스를 전달시키는 방법을 사용할 수도 있습니다.

이 트레이드오프 사이에서 적절한 균형을 찾기 위한 괜찮은 방법 중 하나는 `send()` 혹은 `transfer()`를 사용하여 푸시(*push*)를 구현하고, `call.value()()`를 사용하여 풀(*pull*)을 구현하여 [푸시앤풀 메커니즘(*push* and *pull*)](#favor-pull-over-push-for-external-calls)을 사용하는 것입니다.

한 가지 주의할 점은 `send()`나 `transfer()` 함수를 송금 전용으로 사용하는 것이 컨트랙트 자체를 재진입공격으로부터 안전하게 만드는 것이 아니라 단순히 해당 송금에 대해서만 안전하게 만든다는 것입니다.


### 외부 호출 시 에러 처리하기

솔리디티는 주소값에 직접 접근할 수 있도록 `address.call()`, `address.callcode()`, `address.delegatecall()`, 및 `address.send()` 등의 로우-레벨(low-level) 메소드들을 제공합니다. 이 로우-레벨 메소드들은 예외를 발생시키지 않고 만일 예외 상황이 발생했을 경우에는 `false` 값을 반환합니다 반면, 컨트랙트 콜(*contract calls*, e.g., `ExternalContract.doSomething()`)들은 자동으로 예외를 전달합니다. (예로, `doSomething()`에서 예외를 던질 경우 `ExternalContract.doSomething()` 또한 예외를 던집니다.)

로우-레벨 메소드들을 사용할 경우에는, 반드시 호출 결과를 확인해서 호출이 실패하는 경우를 처리해야만 합니다.

```sol
// 나쁜 케이스
someAddress.send(55);
someAddress.call.value(55)(); // 이 것은 남아있는 모든 가스를 전달한 뒤, 결과를 확인하지 않는 것이므로 곱절로 위험합니다.
someAddress.call.value(100)(bytes4(sha3("deposit()"))); // deposit 호출이 예외를 발생시킨다면 로우-레벨 메소드 call()은 false를 반환하기만 하고 트랜잭션은 무효화(revert)되지 않을 것입니다

// 좋은 케이스
if(!someAddress.send(55)) {
    // 실패하는 코드
}

ExternalContract(someAddress).deposit.value(100);
```


### 외부호출에 푸쉬(*push*)보다는 풀(*pull*)을 사용하세요

외부호출은 의도치 않게 혹은 고의로 실패할 가능성이 있습니다. 이러한 실패들로 인한 피해를 최소화하기 위해서는 각각의 외부 호출들을 호출 수신자가 시작할 수 있는 각 트랜잭션별로 격리하는 것이 좋습니다. 이 것은 특히 지불 등과 같이 사용자들의 펀드를 직접 업데이트 해주기(push)보다는 사용자가 직접 출금해가도록(pull) 하는 것이 나은 상황 등에 적절한 방법입니다.(이런 방법은 [가스 한도와 관련된 문제들](./known_attacks#dos-with-block-gas-limit)의 발생을 줄여줍니다). 즉, 한 트랜잭션에서 `send()` 호출을 여러개 묶어서 사용하면 안됩니다.

```sol
// 나쁜 케이스
contract auction {
    address highestBidder;
    uint highestBid;

    function bid() payable {
        require(msg.value >= highestBid);

        if (highestBidder != 0) {
            highestBidder.transfer(highestBid); // 이 호출이 지속적으로 실패한다면, 아무도 경매에 참가할 수 없습니다
        }

       highestBidder = msg.sender;
       highestBid = msg.value;
    }
}

// 좋은 케이스
contract auction {
    address highestBidder;
    uint highestBid;
    mapping(address => uint) refunds;

    function bid() payable external {
        require(msg.value >= highestBid);

        if (highestBidder != 0) {
            refunds[highestBidder] += highestBid; // 유저가 환불해갈 수 있는 금액을 기록합니다
        }

        highestBidder = msg.sender;
        highestBid = msg.value;
    }

    function withdrawRefund() external {
        uint refund = refunds[msg.sender];
        refunds[msg.sender] = 0;
        msg.sender.transfer(refund);
    }
}
```

## 컨트랙트의 잔고가 무조건 0으로 생성된다고 가정하지 마세요

공격자는 컨트랙트가 생성되기 이전에 해당 컨트랙트의 주소에 이더를 보내놓을 수 있습니다. 따라서 컨트랙트의 시작 상태의 잔고가 0이라고 가정해서는 절대로 안됩니다. 자세한 내용을 확인하려면 다음 [깃허브 이슈issue 61](https://github.com/ConsenSys/smart-contract-best-practices/issues/61)를 확인해주세요

## 체인 상에 기록된 데이터는 완전 공개되어 있습니다

많은 어플리케이션들이 올바른 동작을 위해서는 체인상에 등록한 데이터가 특정 시점까지 비공개로 유지되는 것을 필요로 합니다. 게임(예: 블록 체인 가위바위보) 및 경매 메커니즘 (예: 밀봉 형태의 2차 가격 경매 등)은 비공개를 필요로 하는 예시의 주요 항목들이빈다. 혹은 프라이버시 이슈를 가지고 있는 어플리케이션을 개발한다면 유저가 정보를 너무 쉽게 공개하지 못하도록 해야 합니다.

예시:

* 가위바위보에서는 두 플레이어가 모두 첫 번째로 낼 패의 해쉬 값을 제출합니다. 그리고 둘 모두 해쉬를 제출하고 나면, 원래 값을 제출합니다. 원래 값을 제출했을 때, 기 제출한 해쉬값과 맞지 않으면 게임이 성립되지 않습니다.
* 경매에서는, 1단계로 플레이어들이 경매 값(당연히, 경매 값보다 예금이 더 많은 상태여야 합니다)의 해쉬값을 제출하도록 합니다. 그리고 2단계로 실제 경매값을 제출하도록 합니다.
* 난수 생성기에 의존적인 어플리케이션을 개발하는 경우, 순서는 항상 (1) 플레이어가 값을 제출하고 (2) 난수를 생성하고 (3) 플레이어가 값을 지불하는 순서가 되어야 합니다.
* When developing an application that depends on a random number generator, the order should always be (1) players submit moves, (2) random number generated, (3) players paid out. The method by which random numbers are generated is itself an area of active research; current best-in-class solutions include Bitcoin block headers (verified through http://btcrelay.org), hash-commit-reveal schemes (ie. one party generates a number, publishes its hash to "commit" to the value, and then reveals the value later) and [RANDAO](http://github.com/randao/randao).
* If you are implementing a frequent batch auction, a hash-commit scheme is also desirable.

## In 2-party or N-party contracts, beware of the possibility that some participants may "drop offline" and not return

Do not make refund or claim processes dependent on a specific party performing a particular action with no other way of getting the funds out. For example, in a rock-paper-scissors game, one common mistake is to not make a payout until both players submit their moves; however, a malicious player can "grief" the other by simply never submitting their move - in fact, if a player sees the other player's revealed move and determines that they lost, they have no reason to submit their own move at all. This issue may also arise in the context of state channel settlement. When such situations are an issue, (1) provide a way of circumventing non-participating participants, perhaps through a time limit, and (2) consider adding an additional economic incentive for participants to submit information in all of the situations in which they are supposed to do so.

## Solidity specific recommendations

### <!-- -->

The following recommendations are specific to Solidity, but may also be instructive for developing smart contracts in other languages.

## Enforce invariants with `assert()`

An assert guard triggers when an assertion fails - such as an invariant property changing. For example, the token to ether issuance ratio, in a token issuance contract, may be fixed. You can verify that this is the case at all times with an `assert()`. Assert guards should often be combined with other techniques, such as pausing the contract and allowing upgrades. (Otherwise, you may end up stuck, with an assertion that is always failing.)

Example:

```sol
contract Token {
    mapping(address => uint) public balanceOf;
    uint public totalSupply;

    function deposit() public payable {
        balanceOf[msg.sender] += msg.value;
        totalSupply += msg.value;
        assert(this.balance >= totalSupply);
    }
}
```

Note that the assertion is *not* a strict equality of the balance because the contract can be [forcibly sent ether](#remember-that-ether-can-be-forcibly-sent-to-an-account) without going through the `deposit()` function!


## Use `assert()` and `require()` properly

In Solidity 0.4.10 `assert()` and `require()` were introduced. `require(condition)` is meant to be used for input validation, which should be done on any user input, and reverts if the condition is false. `assert(condition)` also reverts if the condition is false but should be used only for invariants: internal errors or to check if your contract has reached an invalid state. Following this paradigm allows formal analysis tools to verify that the invalid opcode can never be reached: meaning no invariants in the code are violated and that the code is formally verified.

## Beware rounding with integer division

All integer division rounds down to the nearest integer. If you need more precision, consider using a multiplier, or store both the numerator and denominator.

(In the future, Solidity will have a fixed-point type, which will make this easier.)

```sol
// bad
uint x = 5 / 2; // Result is 2, all integer divison rounds DOWN to the nearest integer
```

Using a multiplier prevents rounding down, this multiplier needs to be accounted for when working with x in the future:

```sol
// good
uint multiplier = 10;
uint x = (5 * multiplier) / 2;
```

Storing the numerator and denominator means you can calculate the result of `numerator/denominator` off-chain:
```sol
// good
uint numerator = 5;
uint denominator = 2;
```

## Remember that Ether can be forcibly sent to an account

Beware of coding an invariant that strictly checks the balance of a contract.

An attacker can forcibly send wei to any account and this cannot be prevented (not even with a fallback function that does a `revert()`).

The attacker can do this by creating a contract, funding it with 1 wei, and invoking
`selfdestruct(victimAddress)`.  No code is invoked in `victimAddress`, so it
cannot be prevented.

## Be aware of the tradeoffs between abstract contracts and interfaces

Both interfaces and abstract contracts provide one with a customizable and re-usable approach for smart contracts. Interfaces, which were introduced in Solidity 0.4.11, are similar to abstract contracts but cannot have any functions implemented. Interfaces also have limitations such as not being able to access storage or inherit from other interfaces which generally makes abstract contracts more practical. Although, interfaces are certainly useful for designing contracts prior to implementation. Additionally, it is important to keep in mind that if a contract inherits from an abstract contract it must implement all non-implemented functions via overriding or it will be abstract as well.

## Keep fallback functions simple

[Fallback functions](http://solidity.readthedocs.io/en/latest/contracts.html#fallback-function) are called when a contract is sent a message with no arguments (or when no function matches), and only has access to 2,300 gas when called from a `.send()` or `.transfer()`. If you wish to be able to receive Ether from a `.send()` or `.transfer()`, the most you can do in a fallback function is log an event. Use a proper function if a computation or more gas is required.

```sol
// bad
function() payable { balances[msg.sender] += msg.value; }

// good
function deposit() payable external { balances[msg.sender] += msg.value; }

function() payable { require(msg.data.length == 0); LogDepositReceived(msg.sender); }
```

## Check data length in fallback functions

Since the [fallback functions](http://solidity.readthedocs.io/en/latest/contracts.html#fallback-function) is not only called for plain ether transfers (without data) but also when no other function matches, you should check that the data is empty if the fallback function is intended to be used only for the purpose of logging received Ether. Otherwise, callers will not notice if your contract is used incorrectly and functions that do not exist are called.

```sol
// bad
function() payable { LogDepositReceived(msg.sender); }

// good
function() payable { require(msg.data.length == 0); LogDepositReceived(msg.sender); }
```

## Explicitly mark visibility in functions and state variables

Explicitly label the visibility of functions and state variables. Functions can be specified as being `external`, `public`, `internal` or `private`. Please understand the differences between them, for example, `external` may be sufficient instead of `public`. For state variables, `external` is not possible. Labeling the visibility explicitly will make it easier to catch incorrect assumptions about who can call the function or access the variable.

```sol
// bad
uint x; // the default is internal for state variables, but it should be made explicit
function buy() { // the default is public
    // public code
}

// good
uint private y;
function buy() external {
    // only callable externally
}

function utility() public {
    // callable externally, as well as internally: changing this code requires thinking about both cases.
}

function internalAction() internal {
    // internal code
}
```

## Lock pragmas to specific compiler version

Contracts should be deployed with the same compiler version and flags that they have been tested the most with. Locking the pragma helps ensure that contracts do not accidentally get deployed using, for example, the latest compiler which may have higher risks of undiscovered bugs. Contracts may also be deployed by others and the pragma indicates the compiler version intended by the original authors.

```sol
// bad
pragma solidity ^0.4.4;


// good
pragma solidity 0.4.4;
```

### Exception

Pragma statements can be allowed to float when a contract is intended for consumption by other developers, as in the case with contracts in a library or EthPM package. Otherwise, the developer would need to manually update the pragma in order to compile locally.

## Differentiate functions and events

Favor capitalization and a prefix in front of events (we suggest *Log*), to prevent the risk of confusion between functions and events. For functions, always start with a lowercase letter, except for the constructor.

```sol
// bad
event Transfer() {}
function transfer() {}

// good
event LogTransfer() {}
function transfer() external {}
```

## Prefer newer Solidity constructs

Prefer constructs/aliases such as `selfdestruct` (over `suicide`) and `keccak256` (over `sha3`).  Patterns like `require(msg.sender.send(1 ether))` can also be simplified to using `transfer()`, as in `msg.sender.transfer(1 ether)`.

## Be aware that 'Built-ins' can be shadowed

It is currently possible to [shadow](https://en.wikipedia.org/wiki/Variable_shadowing) built-in globals in Solidity. This allows contracts to override the functionality of built-ins such as `msg` and `revert()`. Although this [is intended](https://github.com/ethereum/solidity/issues/1249), it can mislead users of a contract as to the contract's true behavior.

```sol
contract PretendingToRevert {
    function revert() internal constant {}
}

contract ExampleContract is PretendingToRevert {
    function somethingBad() public {
        revert();
    }
}
```

Contract users (and auditors) should be aware of the full smart contract source code of any application they intend to use.

## Avoid using `tx.origin`

Never use `tx.origin` for authorization, another contract can have a method which will call your contract (where the user has some funds for instance) and your contract will authorize that transaction as your address is in `tx.origin`.

```
pragma solidity 0.4.18;

contract MyContract {

    address owner;

    function MyContract() public {
        owner = msg.sender;
    }

    function sendTo(address receiver, uint amount) public {
        require(tx.origin == owner);
        receiver.transfer(amount);
    }

}

contract AttackingContract {

    MyContract myContract;
    address attacker;

    function AttackingContract(address myContractAddress) public {
        myContract = MyContract(myContractAddress);
        attacker = msg.sender;
    }

    function() public {
        myContract.sendTo(attacker, msg.sender.balance);
    }

}
```

You should use `msg.sender` for authorization (if another contract calls your contract `msg.sender` will be the address of the contract and not the address of the user who called the contract).

You can read more about it here: [Solidity docs](https://solidity.readthedocs.io/en/develop/security-considerations.html#tx-origin)

Besides the issue with authorization, there is a chance that `tx.origin` will be removed from the Ethereum protocol in the future, so code that uses `tx.origin` won't be compatible with future releases [Vitalik: 'Do NOT assume that tx.origin will continue to be usable or meaningful.'](https://ethereum.stackexchange.com/questions/196/how-do-i-make-my-dapp-serenity-proof/200#200)

It's also worth mentioning that by using `tx.origin` you're limiting interoperability between contracts because the contract that uses tx.origin cannot be used by another contract as a contract can't be the `tx.origin`.

## Timestamp Dependence

There are three main considerations when using a timestamp to execute a critical function in a contract, especially when actions involve fund transfer.

### Gameability

Be aware that the timestamp of the block can be manipulated by a miner. Consider this [contract](https://etherscan.io/address/0xcac337492149bdb66b088bf5914bedfbf78ccc18#code):

```sol

uint256 constant private salt =  block.timestamp;

function random(uint Max) constant private returns (uint256 result){
    //get the best seed for randomness
    uint256 x = salt * 100/Max;
    uint256 y = salt * block.number/(salt % 5) ;
    uint256 seed = block.number/3 + (salt % 300) + Last_Payout + y;
    uint256 h = uint256(block.blockhash(seed));

    return uint256((h / x)) % Max + 1; //random number between 1 and Max
}
```

When the contract uses the timestamp to seed a random number, the miner can actually post a timestamp within 30 seconds of the block being validating, effectively allowing the miner to precompute an option more favorable to their chances in the lottery. Timestamps are not random and should not be used in that context.

### *30-second Rule*
A general rule of thumb in evaluating timestamp usage is:
#### If the contract function can tolerate a [30-second](https://ethereum.stackexchange.com/questions/5924/how-do-ethereum-mining-nodes-maintain-a-time-consistent-with-the-network/5931#5931) drift in time, it is safe to use `block.timestamp`
If the scale of your time-dependent event can vary by 30-seconds and maintain integrity, it is safe to use a timestamp. This includes things like ending of auctions, registration periods, etc.

### Caution using `block.number` as a timestamp

When a contract creates an `auction_complete` modifier to signify the end of a token sale such as [so]((https://github.com/SpankChain/old-sc_auction/blob/master/contracts/Auction.sol))
```sol
modifier auction_complete {
    require(auctionEndBlock <= block.number     ||
          currentAuctionState == AuctionState.success ||
          currentAuctionState == AuctionState.cancel)
        _;}
```
`block.number` and *[average block time](https://etherscan.io/chart/blocktime)* can be used to estimate time as well, but this is not future proof as block times may change (such as [fork reorganisations](https://blog.ethereum.org/2015/08/08/chain-reorganisation-depth-expectations/) and the [difficulty bomb](https://github.com/ethereum/EIPs/issues/649)). In a sale spanning days, the 12-minute rule allows one to construct a more reliable estimate of time.

## Multiple Inheritance Caution

When utilizing multiple inheritance in Solidity, it is important to understand how the compiler composes the inheritance graph.

```sol

contract Final {
    uint public a;
    function Final(uint f) public {
        a = f;
    }
}

contract B is Final {
    int public fee;

    function B(uint f) Final(f) public {
    }
    function setFee() public {
        fee = 3;
    }
}

contract C is Final {
    int public fee;

    function C(uint f) Final(f) public {
    }
    function setFee() public {
        fee = 5;
    }
}

contract A is B, C {
  function A() public B(3) C(5) {
      setFee();
  }
}
```
When A is deployed, the compiler will *linearize* the inheritance from left to right, as:

**C -> B -> A**

The consequence of the linearization will yield a `fee` value of 5, since C is the most derived contract. This may seem obvious, but imagine scenarios where C is able to shadow crucial functions, reorder boolean clauses, and cause the developer to write exploitable contracts. Static analysis currently does not raise issue with overshadowed functions, so it must be manually inspected.

For more on security and inheritance, check out this [article](https://pdaian.com/blog/solidity-anti-patterns-fun-with-inheritance-dag-abuse/)

To help contribute, Solidity's Github has a [project](https://github.com/ethereum/solidity/projects/9#card-8027020) with all inheritance-related issues.

## Deprecated/historical recommendations

These are recommendations which are no longer relevant due to changes in the protocol or improvements to solidity. They are recorded here for posterity and awareness.

### Beware division by zero (Solidity < 0.4)

Prior to version 0.4, Solidity [returns zero](https://github.com/ethereum/solidity/issues/670) and does not `throw` an exception when a number is divided by zero. Ensure you're running at least version 0.4.
