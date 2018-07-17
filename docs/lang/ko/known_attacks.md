다음은 꼭 알고 있어야 하는 알려진 공격사례들과 이를 방어하기 위한 스마트 컨트랙트 작성법입니다.

## 경쟁 조건<sup><a href='#footnote-race-condition-terminology'>\*</a></sup>

외부 컨트랙트를 호출할 때 주요 위험중 하나는 제어 흐름을 외부 컨트랙트가 가져갈 수 있다는 것입니다. 그리고 여러분의 컨트랙트의 의도치 않은 방향으로 데이터를 변경할 수 있게 됩니다. 이러한 종류의 버그는 여러가지 형태를 가질 수 있고, DAO 사태를 촉발한 두 주요 버그들이 바로 이 종류였습니다.

### 재진입

이 버그가 알려진 첫 버전은 첫 함수 호출이 종료되기 이전에, 반복적인 실행 요청이 가능한 함수를 포함하고 있었습니다. 이 것은 함수의 다른 함수 호출이 파괴적인 방법으로 상호작용할 수 있게 합니다.

```sol
// 안전하지 않음
mapping (address => uint) private userBalances;

function withdrawBalance() public {
    uint amountToWithdraw = userBalances[msg.sender];
    require(msg.sender.call.value(amountToWithdraw)()); // 여기에서 호출자의 코드가 실행되고 출금을 계속할 수 있습니다
    userBalances[msg.sender] = 0;
}
```

유저의 잔고가 함수가 끝나기 전까지 0으로 설정되지 않기 때문에, 두번째 그리고 그 이후에도 함수 실행요청은 계속 성공합니다. 그리고 잔고를 계속 출금할 수 있습니다. 이와 아주 유사한 버그가 DAO 공격의 취약점이었습니다.

주어진 예시를 보면, 이런 문제를 방지하는 가장 좋은 방법은 [`call.value()()` 대신 `send()`](./recommendations#send-vs-call-value)를 사용하는 것입니다. 이 방법은 외부 코드가 실행되는 것을 막아줍니다.

하지만, 외부 호출을 제거할 수 없다면, 이런 공격을 막을 수 있는 차선책은 내부적인 작업들이 모두 완료되기 전까지는 외부 함수 호출을 하지 않도록 하는 것입니다:

```sol
mapping (address => uint) private userBalances;

function withdrawBalance() public {
    uint amountToWithdraw = userBalances[msg.sender];
    userBalances[msg.sender] = 0;
    require(msg.sender.call.value(amountToWithdraw)()); // 유저의 잔고가 이미 0입니다. 따라서 이후의 함수 실행은 출금을 진행할 수 없습니다
}
```

유의할 것은, `withdrawBalance()`를 실행하는 다른 함수를 이미 가지고 있을 경우, 해당 함수 또한 같은 공격의 대상이 될 수 있다는 것입니다. 따라서 신뢰할 수 없는 컨트랙트를 실행하는 모든 함수를 반드시 신뢰할 수 없는 것으로 취급해야 합니다. 아래에 가능한 해결책들에 대한 추가 논의가 있습니다.

### 함수간 경쟁 조건

공격자는 두 가지 다른 함수를 같은 상태를 가지게 함을 통해서 유사한 공격을 진행할 수 있습니다.

```sol
// 안전하지 않음
mapping (address => uint) private userBalances;

function transfer(address to, uint amount) {
    if (userBalances[msg.sender] >= amount) {
       userBalances[to] += amount;
       userBalances[msg.sender] -= amount;
    }
}

function withdrawBalance() public {
    uint amountToWithdraw = userBalances[msg.sender];
    require(msg.sender.call.value(amountToWithdraw)()); // 여기에서 호출자의 코드가 실행될 수 있고 transfer()를 호출할 수 있음
    userBalances[msg.sender] = 0;
}
```

이 예시에서는, `withdrawBalance` 함수 내에서 외부 호출에 의해 공격자의 코드를 실행시킬 수 있고 이 때 공격자는 `transer()` 함수를 호출할 수 있습니다. 따라서 해당 공격자의 잔고는 0으로 설정되지 않고, 이미 출금하였더라도 계속해서 토큰을 송금할 수 있습니다. 이 취약점은 또한 DAO 공격에서 사용되었습니다.

앞선 것과 같은 방식의 해결책들을 사용해야 합니다. 물론 같은 위험성을 포함합니다. 또한 이 예시에서는 두 함수 모두 같은 컨트랙트 안에 있었지만, 이런 버그는 컨트랙트 간 상태를 공유한다면 여러 컨트랙트를 넘나들며 발생할 수 있음을 유의해야 합니다.

### 경쟁 조건 솔루션들의 함정

경쟁 조건이 함수 간, 심지어는 컨트랙트 간에도 일어날 수 있으므로 재진입을 방지하는 솔루션만으로는 충분하지 않습니다.

대신, 우리는 모든 내부 작업을 먼저 끝내고 그 다음 외부 함수를 호출하는 것을 추천하고 있습니다. 이 규칙을 잘 따를경우, 경쟁조건을 회피할 수 있습니다. 하지만 외부함수를 너무 빨리 호출하는 것 뿐만 아니라 외부 함수를 호출하는 함수들을 호출하는 것 또한 피해야만 합니다. 다음은 안전하지 못한 예입니다:

```sol
// 안전하지 않음
mapping (address => uint) private userBalances;
mapping (address => bool) private claimedBonus;
mapping (address => uint) private rewardsForA;

function withdraw(address recipient) public {
    uint amountToWithdraw = userBalances[recipient];
    rewardsForA[recipient] = 0;
    require(recipient.call.value(amountToWithdraw)());
}

function getFirstWithdrawalBonus(address recipient) public {
    require(!claimedBonus[recipient]); // 각 수신자는 보너스를 한 번만 요청할 수 있습니다

    rewardsForA[recipient] += 100;
    withdraw(recipient); // 여기서 호출자는 getFirstWithdrawalBonus를 다시 호출할 수 있습니다
    claimedBonus[recipient] = true;
}
```

`getFirstWithdrawalBonus()`가 외부 컨트랙트를 직접 호출하지는 않지만, `withdraw()` 함수 내에 담긴 호출은 경쟁 조건으로 내몰기에 충분합니다. 그렇기 때문에 `withdraw()` 함수 또한 신뢰할 수 없는 함수로 취급해야 합니다.

```sol
mapping (address => uint) private userBalances;
mapping (address => bool) private claimedBonus;
mapping (address => uint) private rewardsForA;

function untrustedWithdraw(address recipient) public {
    uint amountToWithdraw = userBalances[recipient];
    rewardsForA[recipient] = 0;
    require(recipient.call.value(amountToWithdraw)());
}

function untrustedGetFirstWithdrawalBonus(address recipient) public {
    require(!claimedBonus[recipient]); // 수신자들은 한 번만 보너스를 요청할 수 있습니다

    claimedBonus[recipient] = true;
    rewardsForA[recipient] += 100;
    untrustedWithdraw(recipient); // claimedBonus가 참으로 설정되어 재진입이 불가능합니다
}
```

재진입이 불가능하게 만드는 수정 사항과 더불어, [신뢰할 수 없는 함수가 표시](./recommendations#mark-untrusted-contracts) 되었습니다. 이 패턴은 모든 수준에서 계속되어야 합니다: `untrustedGetFirstWithdrawalBonus()` 함수가 `untrustedWithdraw()`를 호출하고 이 는 외부 컨트랙트를 호출하므로 `untrustedGetFirstWithdrawalBonus()` 또한 안전하지 않은 것으로 다뤄야 합니다.

추천되는 또다른 해결책은 [뮤텍스(mutex)](https://en.wikipedia.org/wiki/Mutual_exclusion)입니다. 뮤텍스는 특정 상황을 "락(lock)"시키고 "락"의 소유자를 통해서만 수정될 수 있게 합니다. 간단한 예시는 다음과 같습니다:

```sol
// Note: 이것은 초보적인 예시입니다. 뮤텍스는 특히 중요한 로직 및 공유된 상태가 있을 때 유용하게 쓰입니다.
mapping (address => uint) private balances;
bool private lockBalances;

function deposit() payable public returns (bool) {
    require(!lockBalances);
    lockBalances = true;
    balances[msg.sender] += msg.value;
    lockBalances = false;
    return true;
}

function withdraw(uint amount) payable public returns (bool) {
    require(!lockBalances && amount > 0 && balances[msg.sender] >= amount);
    lockBalances = true;

    if (msg.sender.call(amount)()) { // 보통 안전하지 않은 경우이지만, 뮤텍스로 인해 안전해졌습니다
      balances[msg.sender] -= amount;
    }

    lockBalances = false;
    return true;
}
```

사용자가 `withdraw()`를 첫 호출이 끝나기 전에 반복해서 호출하려 시도할 때, 락(lock)이 아무 것도 하지 못하게 방어해줍니다. 이 것은 효과적인 방법이지만 다양한 컨트랙트가 상호작용하는 경우에는 다소 곤란해질 수 있습니다. 다음 상황은 안전하지 않은 상황입니다:

```sol
// 안전하지 않음
contract StateHolder {
    uint private n;
    address private lockHolder;

    function getLock() {
        require(lockHolder == 0);
        lockHolder = msg.sender;
    }

    function releaseLock() {
        require(msg.sender == lockHolder);
        lockHolder = 0;
    }

    function set(uint newState) {
        require(msg.sender == lockHolder);
        n = newState;
    }
}
```

공격자가 `getLock()`을 호출했고 `releaseLock()`을 호출하지 않았습니다. 이 경우 컨트랙트는 영원히 잠겨 교착상태에 빠져버립니다. 그리고 더 이상 아무런 변화를 받아들일 수 없게 됩니다ㅣ. 뮤텍스를 경쟁조건을 방지하기 위해 프로젝트에 사용하는 경우에는 락이 요청된 뒤 해제되지 않는 상황이 발생할 수 없음을 확실하게 신경써서 만들어야 합니다.(뮤텍스를 사용한 프로그래밍에서는 데드락 혹은 라이브락등과 같은 문제가 발생할 가능성이 있습니다. 뮤텍스를 사용할 것이라면, 뮤텍스에 관련된 많은 자료들을 참고해야 합니다.)

<a name='footnote-race-condition-terminology'></a>

<div style='font-size: 80%; display: inline;'>* 일부는 이더리움은 현재 완벽하게 병렬적이지 않기 때문에 <i>경쟁 조건</i>이라는 용어를 사용하는 것에 반대합니다. 하지만 논리적으로는 확실하게 자원에 대해 경쟁하는 절차를 지니는 기본 기능들이 있기 때문에, 같은 종류의 위험이 존재하고 같은 종류의 솔루션들이 동작합니다.</div>

## 트랜잭션 순서 의존성(Transaction-Ordering Dependence: TOD)과 프론트 러닝(Front Running)

상기 예시들은 공격자가 악의적인 코드를 *하나의 트랜잭션 내에* 담아서 실행하는 경쟁 조건들에 대한 것들이었습니다. 다음은 한 블록 내에 포함된 *트랜잭션들의 순서* 를 바꾸는 것이 쉽다는 것을 활용한 블록체인 고유의 경쟁조건에 대한 예시들입니다.


트랜잭션은 멤풀(mempool)에 잠시 있으므로, 해당 트랜잭션이 블록에 포함되기 이전에 어떤 액션이 일어날 지 알 수 있습니다. 이 것은 탈중앙화된 마켓 등에서 골치아파질 수 있습니다. 토큰을 사고자 하는 거래를 볼 수 있고 또 다른 트랜잭션이 포함되기 이전에 순서를 변경할 수 있기 때문입니다. 이 문제를 방어하는 것은 특정한 계약에 대해서만 공격이 이루어질 것이므로 방어하기 어렵습니다. 따라서 예시로 마켓에서는 일괄 경매를 진행하는 것이 더 나을 것입니다(이는 너무 고빈도 트레이딩을 방지해주기도 합니다). 또 다른 방법은 선커밋(pre-commit) 방식을 사용하는 것입나다("자세한 건 나중에 알려줄게, 먼저 해쉬값 받아").

## 타임스탬프 의존성

블록의 타임스탬프는 채굴자에 의해 조작 가능하다는 것을 유의하고, 모든 직간접적인 타임스탬프 사용에 대해 숙고해봐야 합니다.

타임스탬프 의존성과 관련된 설계를 하고 있는 경우, 추천사항 항목에서 이에 관한 [추천사항](./recommendations/#timestamp-dependence)을 확인하세요.

## 정수 오버플로(Overflow) 및 언더플로(Underflow)

단순한 토큰 송금 상황을 보겠습니다:

```sol
mapping (address => uint256) public balanceOf;

// 안전하지 않음
function transfer(address _to, uint256 _value) {
    /* 송금자가 잔고가 있는지 확인 */
    require(balanceOf[msg.sender] >= _value);
    /* 잔고에서 빼고 더함 */
    balanceOf[msg.sender] -= _value;
    balanceOf[_to] += _value;
}

// 안전함
function transfer(address _to, uint256 _value) {
    /* 송금자가 잔고가 있는지 확인하고 오버플로를 체크 */
    require(balanceOf[msg.sender] >= _value && balanceOf[_to] + _value >= balanceOf[_to]);

    /* 잔고에서 빼고 더함 */
    balanceOf[msg.sender] -= _value;
    balanceOf[_to] += _value;
}
```

밸런스가 uint(2~256) 값의 최대치에 도달했을 때, 0으로 값이 순환합니다. 이 것을 확인해야 합니다. 이 문제는 구현에 따라 달라지기도 할 겁니다. uint값이 큰 수를 만나게 될 지 아닐 지, uint 변수가 상태를 어떻게 바꾸는지 그리고 해당 변화를 만드는 주체가 누구인지 생각해보세요. 사용자가 uint 값을 업데이트하는 모든 함수를 호출할 수 있다면 이 공격에 취약해 질 겁니다. 해당 변수를 바꾸는 것이 관리자만 가능하다면 좀 더 안전할 겁니다. 사용자가 매 번 1밖에 수를 증가시킬 수 없다면 limit에 도달할 수 있는 방법이 없을 것이므로 아마 안전할 것입니다.

이 것은 언드폴로에도 마찬가지로 적용됩니다. uint값이 0보다 작아질 수 있도록 만들어졌으면 언더플로가 발생할 수 있고 해당 변수는 최댓값을 가지게 될 것입니다.

uint8, uint16, uint24 등의 작은 크기의 자료형을  사용하는 것을 주의하십시오. 이러한 자료형들은 더욱 쉽게 최댓값에 도달할 수 있습니다.

[여기](https://github.com/ethereum/solidity/issues/796#issuecomment-253578925)에서 오버플로 및 언더플로에 관한 20개 가량의 케이스에 대해서 확인하세요.

### 언더플로 심화: 스토리지 조작
2017년 솔리디티 언더핸드 코딩 대회(Underhanded Solidity Coding Contest)에서 [Doug Hoyte의 제출물](https://github.com/Arachnid/uscc/tree/master/submissions-2017/doughoyte)은 [극찬](http://u.solidity.cc/)을 받았습니다. 출발점이 상당히 흥미롭습니다. C와 같은 형태의 언더플로가 솔리디티 스토리지에 어떤 영향을 끼칠 것인가에 대한 것입니다. 여기 간단한 버전의 코드가 있습니다:

```sol
contract UnderflowManipulation {
    address public owner;
    uint256 public manipulateMe = 10;
    function UnderflowManipulation() {
        owner = msg.sender;
    }

    uint[] public bonusCodes;

    function pushBonusCode(uint code) {
        bonusCodes.push(code);
    }

    function popBonusCode()  {
        require(bonusCodes.length >=0);  // 반복적 표현입니다
        bonusCodes.length--; // 언더플로가 여기에서 일어납니다
    }

    function modifyBonusCode(uint index, uint update)  {
        require(index < bonusCodes.length);
        bonusCodes[index] = update; // bonusCodes.length 보다 적은 아무런 인덱스에나 값을 씁니다
    }

}
```

일반적으로 `manipulateMe` 변수의 위치는 실행 불가능한 `keccak256` 해쉬를 거치지 않고서는 영향을 받을 수 없습니다. 그러나 유동 배열은 순서대로 저장되기 때문에 악의적인 행위자가 `manipulateMe`를 바꾸기 해야하는 것은 단순히 다음과 같습니다:

* `popBonusCode`를 호출하여 언더플로를 발생시킵니다 (Note: 솔리디티에는 [내장 pop함수가 없습니다](https://github.com/ethereum/solidity/pull/3743))
* `manipulateMe` 변수의 스토리지 위치를 계산합니다
* `modifyBonusCode`를 사용해 `manipulateMe`의 값을 수정하고 업데이트 합니다.

실제는, 이런 배열은 즉시 수상한 것으로 간주되지만 복잡한 스마트 컨트랙트 구조에 묻힙니다. 따라서 상수형 변수에 대한 악의적인 공격을 허용하게 될 수도 있습니다.

동적 배열을 사용할 때에는, 컨테이너 데이터 구조가 좋은 사례입니다. 솔리디티의 CRUD [파트1](https://medium.com/@robhitchens/solidity-crud-part-1-824ffa69509a)과 [파트2](https://medium.com/@robhitchens/solidity-crud-part-2-ed8d8b4f74ec) 두 글도 이에 관한 좋은 자료입니다.

<a name="dos-with-unexpected-revert"></a>

## (의도치 않은)revert를 사용한 DoS 공격

다음 간단한 경매 컨트랙트를 살펴보겠습니다:

```sol
// 안전하지 않음
contract Auction {
    address currentLeader;
    uint highestBid;

    function bid() payable {
        require(msg.value > highestBid);

        require(currentLeader.send(highestBid)); // 이전 최고 입찰자에게 환불, 실패시 revert

        currentLeader = msg.sender;
        highestBid = msg.value;
    }
}
```

이전 최고 입찰자에게 화불을 시도할 때, 환불이 실패할 경우 트랜잭션이 무효화됩니다. 이는 악의적인 경매참여자가 환불 절차가 무조건 실패하도록 만들어서 최고 입찰자로 남을 수 있음을 의미합니다. 이렇게 되면 아무도 추가적인 `bid()` 함수를 호출하지 못하게 되고 계속해서 최고 입찰자로 남아있을 수 있습니다. 이에 대한 추천 방안은 위와 같은 방식 대신 [풀 방식 지불 시스템(pull payment system)](./recommendation#favor-pull-over-push-for-external-calls)을 구현하는 것입니다.

다른 예시는 컨트랙트가 사용자들에게 값을 지불하기 위해 배열을 순환할 때(예를 들면 크라우드 펀딩 계약의 서포터들을 위한 것)입니다. 각각의 지불이 성공했는지 확인하려는 것은 당연합니다. 지불이 성공적이지 않았을 경우에는 무효화시켜야 합니다. 여기서 문제는 한 번의 지불에서 문제가 발생할 경우 모든 지불이 무효화 된다는 것입니다. 즉, 한 계정에서 에러가 계속 발생한다면 지불이 절대로 이루어질 수 없게 됩니다.

```sol
address[] private refundAddresses;
mapping (address => uint) public refunds;

// 나쁜 케이스
function refundAll() public {
    for(uint x; x < refundAddresses.length; x++) { // 얼마나 많은 계정이 참가했는지에 따른 정해지지 않은 길이의 순환
        require(refundAddresses[x].send(refunds[refundAddresses[x]])) // 아주 안 좋은 코드, 여기에서 하나만 실패하더라도 모든 자금이 묶여버립니다
    }
}
```

여기서도 마찬가지로 추천되는 방식은 [push보다는 pull을 사용할 것](./recommendations#favor-pull-over-push-for-external-calls)입니다.

## 블록 가스 한도를 활용한 DoS 공격

바로 이전 예시에서 또다른 문제 또한 있었던 것을 눈치챘나요? 모두에게 한 번씩 지불하는 로직은 블록의 가스 한도를 초과하는 상황을 유발합니다. 각 이더리움 블록은 정해진 양의 계산만을 수행할 수 있습니다. 이를 넘어선 계산을 수행하도록 할 경우에는 트랜잭션이 실패하게 됩니다.

이 것은 의도적인 공격 상황이 아니더라도 문제를 발생시킬 수 있습니다. 하지만, 특히 공격자가 필요한 가스를 조작하게 될 수 있다면 더욱 나빠집니다. 이전 예제에서 공격자는 아주 조금씩의 환급을 받는 계정을 많이 추가하여 트랜잭션이 가스 한도를 넘어버리게 만들 수 있습니다. 이를 통해 환급 절차를 일어나지 않도록 만들어버릴 수 있습니다.

이 것이 [push보다는 pull](./recommendations#favor-pull-over-push-for-external-calls)방식을 사용해야 하는 또 다른 이유입니다.

반드시 절대적으로 임의의 크기의 배열을 순환해야 할 경우, 여러 번의 트랜잭션을 사용하여 여러 개의 블록에 포함시키는 경우를 꼭 생각해봐야 합니다. 얼마나 진행되었는지 그리고 진행된 지점부터 진행가능한지 추적하는 것을 필요로 할 것입니다. 예시는 다음과 같습니다:

```sol
struct Payee {
    address addr;
    uint256 value;
}

Payee[] payees;
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

다른 트랜잭션들이 다음 회차의 `payOut()`이 실행되는 동안 다른 트랜잭션들이 실행되어도 아무런 악영향이 없어야 합니다. 이러한 패턴은 반드시 필요할 때만 사용하도록 하세요.

## 컨트랙트에 강제로 이더를 보내는 상황

폴백 함수를 사용하지 않고도 컨트랙트에 강제로 이더를 보내는 것이 가능합니다. 이는 폴백 함수에 중요한 로직을 넣거나 혹은 컨트랙트의 잔고에 기반한 계산을 수행할 때 중요하게 고려해야 하는 요소입니다. 다음 예시를 확인해주세요:

```sol
contract Vulnerable {
    function () payable {
        revert();
    }

    function somethingBad() {
        require(this.balance > 0);
        // 여기서 나쁜 짓을 합니다
    }
}
```

컨트랙트의 로직이 컨트랙트에 대한 지불을 불가능하게 만들고 이를 통해 나쁜 일이 일어나는 것을 막을 수 있을 것처럼 보입니다. 하지만, 컨트랙트에 이더를 강제로 보낼 수 있는 다른 몇가지 방법이 존재합니다. 따라서 이 것은 잔고를 0보다 크게 만들 수 있습니다.

컨트랙트 메소드인 `selfdestruct`는 유저가 남은 이더를 어디로 전송할 것인지 특정할 수 있게 해줍니다. `selfdestruct`는 [폴백 함수를 부르지 않습니다](https://solidity.readthedocs.io/en/develop/security-considerations.html#sending-and-receiving-ether).

컨트랙트의 주소를 [미리 계산](https://github.com/Arachnid/uscc/tree/master/submissions-2017/ricmoo)하고 배포되기 이전에 이더를 보내는 것 또한 가능한 방법입니다.

컨트랙트 개발자는 이더가 컨트랙트에 강제적으로 보내질 수 있다는 사실을 알고 있는 상태로 컨트랙트 로직을 이에 맞게 설계해야만 합니다. 일반적으로 펀딩의 출처를 제한하는 것은 불가능하다고 보면 됩니다.

## 폐기된 과거 공격사례들

현재는 프로토콜의 변화와 솔리디티의 향상으로 더이상 가능하지 않게 된 공격들이 있습니다. 이에 해당하는 것들을 다음 사람들이 잘 파악할 수 있도록 아래에 이전 공격사례들이 기록되어 있습니다.
These are attacks which are no longer possible due to changes in the protocol or improvements to solidity. They are recorded here for posterity and awareness.

### 호출 스택 깊이를 활용한 공격(deprecated)

[EIP 150](https://github.com/ethereum/EIPs/issues/150) 하드포크에 따라서 호출 스택 깊이 제한을 활용한 공격은 더 이상 불가능합니다.<sup><a href='http://ethereum.stackexchange.com/questions/9398/how-does-eip-150-change-the-call-depth-attack'>\*</a></sup> (1024에 이르는 호출 스택을 모두 사용하기 전에 가스가 모두 소모됩니다).
