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
* 난수 생성기에 의존적인 어플리케이션을 개발하는 경우에는 항상 다음과 같이 순서를 사용해야 합니다. (1) 플레이어가 이동값을 제출하고 (2) 난수를 생성하고 (3) 플레이어가 값을 지불합니다. 난수생성 방법에 대한 것은 계속해서 연구중에 있는 주제입니다. 현재까지 중 가장 괜찮은 솔루션은 비트코인 블록 헤더의 방식(http://btcrelay.org 를 통해 검증됨)과 해쉬-커밋-공개 방식(즉, 한 측이 수를 생성하고 그 해쉬 값을 게재한 뒤, 실제 값을 나중에 공개하는 방식) 그리고 [RANDAO](http://github.com/randao/randao)입니다.
* 고빈도 일괄 경매를 구현할 경우에는 해쉬-커밋 방식이 바람직합니다.

## 양자간 혹은 다자간 계약에서, 참가자가 오프라인으로 이탈하고 돌아오지 못하는 상황을 염두에 두어야 합니다

특정 당사자 측에서 어떤 액션을 취해주어야만 환불이 가능하도록 환불, 클레임 절차를 만들어서는 안됩니다. 예를 들어, 가위바위보 게임을 만들 때 보통 할 수 있는 실수는 양 참가자가 모두 패를 제출하기 전에는 게임의 결과를 내지 않는 것이 있습니다. 하지만 이 때 악의적인 참가자는 단지 아무것도 제출하지 않음을 통해 다른 참가자들을 비통한 상황에 몰아버릴 수 있습니다. 한 참가자가 다른 참가자의 패를 확인하였는데 지는 게임이었다면 굳이 패를 제출할 이유가 없습니다. 이 문제는 스테이트 채널(state-channel)에서도 발생합니다. 이런 문제가 발생하는 상황에서는 (1) 시간 제한 등을 통해, 네트워크에 실제로 참가하지않는 참여자를 배제하도록 하고 (2) 기대한 상황대로 잘 참여하여 정보를 제출한 참가자들에게 추가적인 경제적인 보상책을 주는 것에 대해 고려해봐야 합니다.

## 솔리디티를 위한 추천 사항들

### <!-- -->

다음 추천 사항들은 솔리디티에 특화된 것들입니다. 하지만 스마트 컨트랙트를 다른 언어를 사용하여 개발할 때에도 어느정도 지침으로 사용할 수 있습니다.

## `assert()`를 통해 불변성을 강제하세요
## Enforce invariants with `assert()`

단언문 방패(assert guard)는 변경되면 안되는 속성값이 변경되었을 경우처럼 단언문이 실패할 때 발동됩니다. 예를 들어, 토큰 발행 계약에서, 이더에 대한 토큰 발행 비율은 아무래도 고정되어 있어야 할 것입니다. 이 것을 매번 `assert()`를 호출함을 통해서 비율이 고정되어 있음을 검증할 수 있습니다. 단언문 방패는 업데이트 기법 등의 다른 기법들과 잘 결합되어 사용되어야만 합니다. 컨트랙트를 잠시 멈추고 업그레이드를 허용하는 상황등에서 잘못 사용할 경우 단언문이 항상 실패하면서 손쓸 수 없는 상태가 되어버릴 수 있습니다.

예시:

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

여기에서 단언문은 잔고와 공급이 완전히 같을 것을 요구하고 있지 않습니다. 이는 컨트랙트가 `deposit()` 함수를 사용하지 않고도, [강제로 이더를 받을 수 있는 방법](#remember-that-ether-can-be-forcibly-sent-to-an-account)이 있기 때문입니다.

## `assert()`와 `require()`를 적절하게 사용하세요

솔리디티 버전 0.4.10에서 `assert()`와 `require()`가 소개되었습니다. `require(condition)` 표현은 입력값의 유효성을 검사하기 위해 사용됩니다. 이는 모든 사용자 입력 값에 사용되어야만 하고 조건에 문제가 있을 경우 트랜잭션을 무효화 합니다. `assert(condition)` 포현 또한 조건이 안 맞았을 경우 트랜잭션을 무효화 시키지만 불변 속성들에 대해서만 사용되어야 합니다. 따라서 내부적으로 에러가 생겼거나, 컨트랙트가 유효하지 않는 상황에 다다른 것을 의미합니다. 이 패러다임을 따르면, 정형화된 분석 도구들을 사용하여 컨트랙트의 불변 속성들이 잘 유지되고 있는지 또 컨트랙트 코드가 잘 정형화 되어 있는지 등을 확인하여 잘못된 옵코드(opcode)가 없음을 검증할 수 있습니다

## 정수를 나눌 때 반올림에 대해 주의하세요

모든 정수 나눗셈은 가장 가까운 정수로 내림처리됩니다. 더 정확한 계산이 필요할 경우에는 곱셈을 사용해보세요. 혹은 제수와 피제수를 모두 저장하는 것도 방법입니다.

(솔리디티는 향후 이 문제를 더 쉽게 처리할 수 있도록 부동소수형을 지원할 것입니다.)

```sol
// 나쁜 케이스
uint x = 5 / 2; // 결과값이 2입니다. 모든 정수 나눗셈은 가까운 값으로의 "내림" 연산을 진행합니다
```

내림 연산을 피하기 위해서 승수를 사용합니다. 나중에 x를 실제로 사용할 때에는 승수를 고려해서 사용해야 합니다:

```sol
// 좋은 케이스
uint multiplier = 10;
uint x = (5 * multiplier) / 2;
```

제수와 피제수를 같이 저장할 경우, 오프-체인 상에서 결과를 직접 계산할 수 있습니다:

```sol
// 좋은 케이스
uint numerator = 5;
uint denominator = 2;
```

## 이더가 아무 계좌에나 강제적으로 전송될 수 있다는 것을 꼭 명심하세요


컨트랙트의 잔고를 엄격하게 체크하도록 하는 불변 속성을 코딩할 때 주의하십시오.

공격자는 강제로 어느 컨트랙트에게나 웨이(wei)를 전송할 수 있고 이 것을 막을 수는 없습니다(심지어 폴백 함수가 `revert()`를 시행하더라도).

공격자는 계약을 생성하고 1웨이를 넣은 다음 `selfdestruct(victimAddress)`를 호출 할 수 있습니다. `victimAddress`의 아무런 코드도 실행될 수 없으므로 막을 수 없는 공격이 됩니다.

## 추상 컨트랙트와 및 인터페이스 사이의 트레이드오프에 대해 유의하세요

인터페이스와 추상 컨트랙트는 둘 모두 스마트 컨트랙트에 대해 사용자화 가능하고 재사용 가능한 접근 방법을 제공해줍니다. 솔리디티 0.4.11 버전에서 소개된 인터페이스는 추상 컨트랙트와 비슷하지만 함수의 구현체를 가질 수 없습니다. 또한 인터페이스는 스토리지 변수에 접근할 수 없고 또 추상 컨트랙트에서는 더욱 유용한 다른 인터페이스로부터 상속받는 기능을 사용할 수 없습니다. 그렇지만, 인터페이스는 확실히 구현보다 설계를 위해 유용하게 사용될 수 있습니다. 반면 추상 컨트랙트를 상속받는 경우에는, 모든 미 구현된 함수를 오버라이딩을 통해 구현하거나 스스로 추상 컨트랙트가 되어야만 하는 것을 명심하세요.

## 폴백 함수를 단순하게 유지하세요

[폴백 함수](http://solidity.readthedocs.io/en/latest/contracts.html#fallback-function)는 컨트랙트에 아무런 변수 없이 메세지가 도착했을 때 혹은 일치하는 함수가 없을 때 호출 됩니다. 그리고 `.send()`나 `. transer()`를 통해 호출되었을 경우에는 오직 2,300 가스만 주어집니다. 만일 이더를 `.send()`나 `.transfer()`를 통해 수신하도록 만들고자 한다면, 폴백 함수에서 할 수 있는 것은 이벤트 로그를 남기는 것 정도 입니다. 계산이나 가스 소모가 필요한 일은 적절한 다른 함수를 사용해야 합니다.

```sol
// 나쁜 케이스
function() payable { balances[msg.sender] += msg.value; }

// 좋은 케이스
function deposit() payable external { balances[msg.sender] += msg.value; }

function() payable { require(msg.data.length == 0); LogDepositReceived(msg.sender); }
```

## 폴백 함수의 데이터 길이를 체크하세요

[폴백 함수](http://solidity.readthedocs.io/en/latest/contracts.html#fallback-function)는 아무런 데이터 없이 이더 송금을 위해서만 사용되는 것이 아니고 일치하는 함수가 없었을 경우에도 사용될 수 있습니다. 따라서 폴백 함수가 이더를 수신하는 것을 로깅하는 용도로 사용된다면, 데이터가 비어있는지 꼭 확인해야 합니다. 그렇지 않으면, 호출자는 컨트랙트가 잘못 사용되었거나 혹은 존재하지 않는 함수를 잘 못 호출했을 때 이를 알아차릴 수 없습니다.


```sol
// 나쁜 케이스
function() payable { LogDepositReceived(msg.sender); }

// 좋은 케이스
function() payable { require(msg.data.length == 0); LogDepositReceived(msg.sender); }
```

## 명료하게 함수와 상태 변수의 가시성에 대해서 표시해야 합니다

함수와 상태 변수의 가시성에 대해서 명료하게 표시해야 합니다. 함수는 `external`, `public`, `internal` 혹은 `private`으로 분류할 수 있습니다. `public`을 쓰기보다는 `external`로도 충분한 경우 등이 있으므로 각각의 차이점에 대해서 숙지해야 합니다. 상태 변수에 대해서는 `external`을 사용해서는 안됩니다. 가시성을 명료하게 표기하는 것을 통해 함수를 호출하거나 변수에 접근하는 범위에 대해 실수하는 것을 쉽게 잡아낼 수 있습니다.

```sol
// 좋은 예시
uint x; // 상태변수를 위한 기본값은 internal입니다. 하지만 확실하게 명시해야 합니다.
function buy() { // 함수의 기본값은 public입니다.
    // public 함수 코드
}

// 좋은 예시
uint private y;
function buy() external {
    // 외부 계정만 호출 가능합니다
}

function utility() public {
    // 외부 계정도 호출 가능하지만 내부적으로도 호출 가능합니다. 이 코드를 수정하려면 양 쪽의 상황에 대해 모두 생각해봐야만 합니다.
}

function internalAction() internal {
    // internal 함수 코드
}
```

## 프라그마(pragma)를 특정 컴파일러 버전에 대해 고정해야 합니다

컨트랙트는 가장 많이 테스트된 환경과 같은 컴파일러 버전 및 플래그 옵션으로 배포되어야만 합니다. 따라서 프라그마(pragma)를 고정하는 것은 컨트랙트를 의도치 않게 최신 컴파일러 버전 등을 사용해 배포되지 않도록 해줍니다. 최신 컴파일러 버전은 상대적으로 위험하고 발견되지 않은 버그를 포함할 수도 있습니다. 컨트랙트는 다른 사람들에 의해 배포될 수도 있기 때문에 프라그마(pragma)는 원작자의 의도에 따른 컴파일러 버전을 지시해야 합니다.

```sol
// 나쁜 케이스
pragma solidity ^0.4.4;


// 좋은 케이스
pragma solidity 0.4.4;
```

### 예외 처리

라이브러리에 포함된 컨트랙트나 EthPM 패키지와 같은 경우처럼, 다른 개발자이 사용하도록 의도된 컨트랙트의 경우에는 프라그마문에 `^`를 사용해 최신 버전을 허용하도록 할 수 있습니다. 그렇지 않으면, 개발자들이 로컬환경에서 컴파일하기 위해서는 직접 수동으로 프라그마를 수정해야합니다.

## 함수와 이벤트를 차별화해야 합니다

이벤트에는 대문자를 쓰는 것과 접두어를 활용하는 것을 추천합니다(*Log* 라는 접두어를 추천). 이를 통해 함수와 이벤트 간의 혼동을 방지할 수 있습니다. 함수는 생성자(constructor)를 제외하고는 꼭 소문자로 시작해야 합니다.

```sol
// 나쁜 케이스
event Transfer() {}
function transfer() {}

// 좋은 케이스
event LogTransfer() {}
function transfer() external {}
```

## 새로운 솔리디티 API를 사용하세요

`suicide` 보다는 `selfdestruct`, `sha3` 보다는 `keccak256` 등 처럼, 새로운 솔리디티 API를 사용하세요. `require(msg.sender.send(1 ether))` 등의 패턴은 `transfer()`등을 통해 `mesg.sender.transfer(1 ether)`처럼 단순화 시킬 수 있습니다.

## '내장 속성(buit-in)'의 경우 숨김처리가 가능합니다

솔리디티에서 내장 전역 속성(built-in globals)에 대해서는 [숨김 처리(shadow)](https://en.wikipedia.org/wiki/Variable_shadowing)가 가능합니다. 내장 기능인 `msg`나 `revert()` 등의 기능을 오버라이드할 수 있습니다. 이 것은 [의도된 바](https://github.com/ethereum/solidity/issues/1249)이기는 하지만 사용자들이 컨트랙트를 사용할 때 실제 기대한 바와 다르게 동작시킬 위험이 존재합니다.

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

컨트랙트 사용자는 혹은 감사자는 해당 어플리케이션이 사용하고자 하는 전체 스마트 컨트랙트 코드에 대해서 인지하고 있어야 합니다.

## `tx.origin`의 사용을 피하세요

`tx.origin`을 인증의 목적으로 절대 사용하지 마세요. 다른 컨트랙트가 여러분의 컨트랙트(예를 들면 사용자의 자금이 보관된)를 호출하는 메소드를 가질 수 있고, 여러분의 컨트랙트는 해당 트랜잭션의 `tx.origin`을 여러분의 컨트랙트로 인식하기 때문에 잘못된 인증을 할 수 있습니다.

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

인증을 위해서는 반드시 `msg.sender`를 사용해야 합니다(만약 다른 컨트랙트가 여러분의 컨트랙트를 호출하는 경우에는 `msg.sender`는 컨트랙트를 호출한 사용자의 주소가 아니라 호출 컨트랙트의 주소가 됩니다).

이에 대해서는 여기에서 더 확인할 수 있습니다: [솔리디티 문서: tx-origin](https://solidity.readthedocs.io/en/develop/security-considerations.html#tx-origin)

인증과 관련한 문제 때문에, 향후 이더리움 프로토콜에서 `tx.origin`이 제거될 가능성이 있습니다. 따라서 `tx.origin`을 사용하는 코드의 경우에는 향후 릴리즈 버전들과 호환되지 않을 것입니다. [비탈릭: tx.origin이 사용가능하지 않아지거나 의미가 없어질 수 있는 것을 고려해주세요'](https://ethereum.stackexchange.com/questions/196/how-do-i-make-my-dapp-serenity-proof/200#200)

한 가지 알아야 할 것은 `tx.origin`을 사용하는 것은 컨트랙트 간의 상호운용성을 제한한다는 것입니다. 왜냐면 `tx.origin`을 사용하는 컨트랙트는 컨트랙트를 `tx.origin`으로 받을 수 없으므로, 다른 컨트랙트가 사용할 수 없게 됩니다.

## 타임스탬프 의존성

컨트랙트 내부에서 특히 송금이 결합된 경우 등의 중요한 함수를 실행할 때 타임스탬프를 사용하는 경우에는 다음 3 가지 사항을 고려해야 합니다.

### 게임성

블록의 타임스탬프는 채굴자에 의해 조작될 수 있습니다. 다음 [컨트랙트](https://etherscan.io/address/0xcac337492149bdb66b088bf5914bedfbf78ccc18#code)를 확인해 보세요:

```sol

uint256 constant private salt =  block.timestamp;

function random(uint Max) constant private returns (uint256 result){
    // 무작위 씨드 확보
    uint256 x = salt * 100/Max;
    uint256 y = salt * block.number/(salt % 5) ;
    uint256 seed = block.number/3 + (salt % 300) + Last_Payout + y;
    uint256 h = uint256(block.blockhash(seed));

    return uint256((h / x)) % Max + 1; // 1과 최댓값 사이의 난수
}
```

컨트랙트가 난수 생성을 위한 씨드로 타임스탬프를 사용할 때, 채굴자는 검증받는 블록에 대해 타임스탬프를 블록이 검증받는 30초 내에 타임스탬프를 찍을 수 있습니다. 이는 채굴자가 난수 생성 값을 채굴자에게 더 유리하게 사전 계산할 수 있도록 합니다. 따라서 타임스탬프는 랜덤하지 않고 이런 용도로 사용되어서는 안됩니다.

### *30초 룰*

타임스탬프 사용의 유효성을 확인하는 일반적인 규칙은 다음과 같습니다:

#### 컨트랙트 함수가 [30초]((https://ethereum.stackexchange.com/questions/5924/how-do-ethereum-mining-nodes-maintain-a-time-consistent-with-the-network/5931#5931)의 오차를 허용할 때, `block.timestamp`를 사용할 수 있습니다
시간 의존적인 이벤트의 시간 단위가 30초 이상에서 무결성을 유지할 수 있다면, 타임스탬프를 사용할 수 있습니다. 이는 경매의 종료나 등록기간 종료 등에 활용할 수 있습니다.

### `block.number`를 타임스탬프처럼 사용하는 것을 주의하세요

컨트랙트가 토큰 세일의 종료를 나타내고자 [다음과 같이](https://github.com/SpankChain/old-sc_auction/blob/master/contracts/Auction.sol) `autction_complete` 한정자를 생성하는 경우에,
```sol
modifier auction_complete {
    require(auctionEndBlock <= block.number     ||
          currentAuctionState == AuctionState.success ||
          currentAuctionState == AuctionState.cancel)
        _;}
```
`block.number`와 [*평균 블록 생성 시간*](https://etherscan.io/chart/blocktime)은 시간을 추정하는 것에 사용될 수 있긴 하지만 블록 시간이 변경될 수 있고([포크 재조정](https://blog.ethereum.org/2015/08/08/chain-reorganisation-depth-expectations/)등으로 인해) 또 [난이도 폭탄](https://github.com/ethereum/EIPs/issues/649)으로 인해 미래를 확실하게 보장할 수는 없습니다.

## 다중 상속을 주의하세요

솔리디티에서 다중 상속을 활용할 때, 컴파일러가 상속 구조를 어떻게 구성하는지 이해하는 것이 중요합니다.

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
A가 배포될 때, 컴파일러는 상속 구조를 왼쪽에서 오른쪽으로 선형화시킬 겁니다. 마치:

**C -> B -> A** 와 같이 됩니다.

C가 마지막 파생 컨트랙트이므로 선형화의 결과는 `fee` 값을 5라고 할 것입니다. 명확해보일 수도 있지만, C가 중요 함수들을 쉐도잉하거나, 조건절들의 순서를 바꾸거나, 개발자가 악성 컨트랙트를 작성할 수 있게 하는 상황 등을 생각해봅시다. 현재의 정적분석은 오버쉐도우 함수에 대해서 이슈를 제기하지 않게 때문에, 이런 경우들은 반드시 수동으로 검사되어야 합니다.

보안 및 상속과 관련한 더 많은 정보는 다음 [글](https://pdaian.com/blog/solidity-anti-patterns-fun-with-inheritance-dag-abuse/)을 참조하세요.

추가적인 기여를 위해서, 솔리디티의 깃허브 저장소에 모든 상속관련 이슈를 다루는 [프로젝트](https://github.com/ethereum/solidity/projects/9#card-8027020)를 확인해주세요.

## 폐기된 과거 추천 사항들

프로토콜이 바뀌어가고 솔리디티가 개선됨에 따라서 더 이상 사용할 필요가 없는 추천사항들이 있습니다. 이에 해당하는 것들을 다음 사람들이 잘 파악할 수 있도록 아래에 폐기된 추천사항들이 기록되어 있습니다.

### 0으로 나누는 것을 주의하세요(솔리디티 버전 0.4 미만)

버전 0.4 이전에서 솔리디티는 숫자가 0으로 나뉘어 졌을 때 예외를 던지지 않고 [0을 반환](https://github.com/ethereum/solidity/issues/670)했습니다. 따라서 최소한 0.4 버전 이후를 사용하는 것을 확실하게 하세요.
