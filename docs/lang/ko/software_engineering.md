[기본 철학](./general_philosophy) 항목에서 논의했던 것과 같이, 알려진 공격에 대처하는 것만으로는 충분하지 않습니다. 블록체인 프로그램에서는 실패에 대한 비용이 아주 높기 때문에, 소프트웨어를 작성할 때 리스크를 줄이는 쪽으로 생각하는 것에 반드시 적응해야 합니다.

우리가 주창하는 접근방법은 "실패에 대비하기"입니다. 코드가 안전한지 아닌지 완벽하게 미리 아는 것은 불가능합니다. 하지만, 컨트랙트를 설계할 때, 실패했을 경우 유연하게 대처할 수 있도록 설계할 수 있고 이를 통해 피해를 최소화할 수 있습니다. 이번 항목에서는 실패에 대비하는 다양한 기법들에 대해 소개하겠습니다.

Note: 시스템에 새로운 구성요소를 추가할 때 항상 리스크가 존재합니다. 잘못 디자인된 '실패 대비(fail-safe)'는 그 자체로 취약점이 될 수 있습니다. 또한 잘 설계된 다수의 '실패 대비(fail-safe)' 코드 사이의 상호 작용 또한 취약점이 될 수 있습니다. 견고한 시스템을 만들기 위해서는, 각 기법을 컨트랙트에 적용할 때 같이 동작할 경우엔 어떻게 되는지도 생각해보아야 합니다.

### 고장난 컨트랙트의 업그레이드

코드는 에러가 발견되거나 혹은 개선사항이 만들어졌을 때 변경을 요합니다. 버그를 발견했는데 처리할 수 없다면 좋지 않습니다.

스마트 컨트랙트를 위한 효율적인 업그레이드 시스템을 설계하는 것은 활발한 연구 분야이며, 이에 대한 모든 범위의 내용을 이 문서에 담기는 어렵기 때문에 가장 많이 사용되는 두 기본적인 접근방법을 소개합니다. 둘 중 더 단순한 방식은 레지스트리 컨트랙트를 사용하여 해당 컨트랙트가 최신 버전의 컨트랙트의 주소값을 가리키게 하는 것입니다. 컨트랙트 사용자를 위한 더욱 매끄러운 접근 방식은 호출과 데이터를 최신 버전의 컨트랙트에 전달하는 방식입니다.

어떤 방식을 사용하던지, 모듈화하고 구성요소를 잘 분리하는 것은 중요합니다. 이를 통해 코드 변경이 기능을 훼손하거나, 데이터를 잃거나, 포팅을 위해 상당한 양의 비용을 사용하는 일 등이 발생하지 않도록 할 수 있습니다. 특히 복잡한 로직을 데이터 스토리지와 분리해서 기능을 바꾸었을 때 데이터를 모두 변경할 필요가 없도록 하면 좋습니다.

또한 계약 당사자들이 코드를 업그레이드하는 것을 결정할 수 있는 안전한 방법을 가지는 것 또한 대단히 중요합니다. 컨트랙트에 따라, 코드의 변경이 한 신뢰 가능한 참가측으로부터 승인받도록 하거나 전체 이해 당사간의 투표를 통해 결정하는 것도 가능합니다. 이러한 절차가 약간의 시간을 소모하기 때문에 공격 상황 등에서 [긴급 정지 혹은 서킷브레이커 발동](#circuit-breakers-pause-contract-functionality)과 같이 더욱 빠르게 반응할 수 있는 방법에 대해서도 필요할 것입니다.

**예시 1: 레지스트리 컨트랙트를 사용하여 최신 버전의 컨트랙트를 저장**

이 예시에서는 호출이 전달되는 방식이 아니므로 유저가 직접 매번 현재 주소를 불러온 다음 상호작용 해야만 합니다.

```sol
contract SomeRegister {
    address backendContract;
    address[] previousBackends;
    address owner;

    function SomeRegister() {
        owner = msg.sender;
    }

    modifier onlyOwner() {
        require(msg.sender == owner)
        _;
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

이 방식은 두 개의 주요 단점이 있습니다.

1. 사용자가 항상 현재 주소를 확인하고 있어야만 합니다. 그렇지 않은 참가자는 기존 버전의 컨트랙트를 사용하여 실패할 수 있는 리스크를 져야만 합니다.
2. 컨트랙트가 대체되었을 때, 이에 저장된 데이터를 어떻게 다룰 것인가에 대해 생각해야 합니다.

이에 대한 대안으로 컨트랙트가 함수 호출과 데이터를 최신 버전의 컨트랙트로 전달하는 방법이 있습니다:

**예시 22: [`DELEGATECALL`을 사용](http://ethereum.stackexchange.com/questions/2404/upgradeable-contracts)하여 데이터와 함수 호출을 전달하기**

```sol
contract Relay {
    address public currentVersion;
    address public owner;

    modifier onlyOwner() {
        require(msg.sender == owner);
        _;
    }

    function Relay(address initAddr) {
        currentVersion = initAddr;
        owner = msg.sender; // 소유자는 단순한 컨트랙트 소유자가 아니라 multisig를 가진 어떤 컨트랙트가 될 것입니다
    }

    function changeContract(address newVersion) public
    onlyOwner()
    {
        currentVersion = newVersion;
    }

    function() {
        require(currentVersion.delegatecall(msg.data));
    }
}
```

이 방법은 통해 기존의 문제점을 해소할 수 있지만 이 또한 고유의 문제가 있습니다. 데이터를 컨트랙트에 어떠헥 저장할 지 극도로 조심해야만 하는 점입니다. 새로운 컨트랙트가 기존 것과 비교하여 다른 스토리지 레이아웃을 가지고 있을 경우 기존 데이터는 손상되어 버릴 것입니다. 더불어서 이 패턴의 간단한 버전은 함수로부터 값을 반환할 수 없고 오직 전달만 하기 때문에 사용성에 제한이 있습니다. (다음의 [심화된 구현들](https://github.com/ownage-ltd/ether-router)은 인라인 어셈블리 코드를 사용하고 리턴 값 사이즈에 대한 레지스트리를 사용하여 이 문제를 해결하려 시도 중입니다)

어떤 방식을 사용하던지, 컨트랙트를 업그레이드 하는 ㅂ아법을 가지고 있는 것은 중요합니다. 그렇지 않을 경우 필연적으로 버그를 만날 것이고 결국 사용할 수 없게 될 것입니다.

### 써킷브레이커(Circuit Breakers): 컨트랙트의 기능을 중지하기

써킷 브레이커는 특정 조건이 만족될 경우 컨트랙트의 실행을 중지시키고 새로운 에러가 발견되었을 때 유용하게 사용될 수 있습니다. 예를 들어, 버그가 발견되었을 경우 대부분의 액션을 중지하고, 출금만 가능하도록 할 수 있습니다. 또한 특정한 신뢰할 수 있는 단체에 서킷 브레이커를 발동할 수 있도록 권한을 주거나 자동으로 실행될 수 있도록 프로그램적인 규칙을 둘 수도 있습니다.

예시:

```sol
bool private stopped = false;
address private owner;

modifier isAdmin() {
    require(msg.sender == owner);
    _;
}

function toggleContractActive() isAdmin public {
    // 추가적인 한정자(modifier)를 사용하여 컨트랙트의 중지를 사용자들의 투표에 의한 액션들에 의해서만 가능하도록 한정지을 수 있습니다
    stopped = !stopped;
}

modifier stopInEmergency { if (!stopped) _; }
modifier onlyInEmergency { if (stopped) _; }

function deposit() stopInEmergency public {
    // 코드
}

function withdraw() onlyInEmergency public {
    // 코드
}
```

### 과속 방지턱(Speed Bumps): 컨트랙트 액션 지연

과속방지턱은 액션들의 실행 속도를 줄여주고 악의적인 액션이 발생했을 때 복구할 시간을 확보할 수 있습니다. 예를 들어 [The DAO]는 DAO를 나누기 위한 요청과 그 발동 조건 사이에 27일 간의 시간을 요구합니다. 이 것은 컨트랙트에 포함된 자금이 복구될 수 있는 가능성을 더욱 높여줍니다. DAO의 경우에서는 과속방지턱에 의해 주어진 시간동안 수행할 특별히 효과적인 액션은 없었지만 다른 기법들과 이를 함께 사용한다면 꽤 효과적일 수 있습니다.

예시:

```sol
struct RequestedWithdrawal {
    uint amount;
    uint time;
}

mapping (address => uint) private balances;
mapping (address => RequestedWithdrawal) private requestedWithdrawals;
uint constant withdrawalWaitPeriod = 28 days; // 4주

function requestWithdrawal() public {
    if (balances[msg.sender] > 0) {
        uint amountToWithdraw = balances[msg.sender];
        balances[msg.sender] = 0; // 우선 단순하게 모두 출금합니다
        // 출금들이 진행 중에 있을 때에는 새로운 예금이 불가능하도록 해야 할 겁니다

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

        require(msg.sender.send(amountToWithdraw));
    }
}
```

### 비율 제한

비율 제한은 상당한 양의 변화를 멈추게 하거나 승인을 요하도록 합니다. 예를 들어, 한 예금자가 특정한 금액만 혹은 전체 예금 대비 특정한 비율에 대해서만 시간 간격을 두고 인출하게 만든다면 (즉, 1일당 최대 100 이더) 이를 넘어서는 추가적인 인출 시도는 실패하거나 특별한 인증을 요구하는 방법입니다. 혹은 특정한 양의 토큰이 시간 비율에 따라 발행되도록 컨트랙트 수준에서 비율 제한을 사용할 수도 있습니다.

[예시](https://gist.github.com/PeterBorah/110c331dca7d23236f80e69c83a9d58c#file-circuitbreaker-sol)

### Contract Rollout

컨트랙트는 상당한 자금을 받게 된다면, 위험에 처하지 않도록 꼭 자금을 받기 이전에 충분하고 긴 기간의 테스트 기간을 거쳐야 합니다.

최소한으로 반드시:

- 100% 혹은 그에 근접한 테스트 커버리지를 달성하는 완전한 테스트수트를 가지고 있어야 합니다
- 테스트 넷을 배포해야 합니다.
- 공개된 테스트넷에 배포하고 충분한 테스트를 짆애하고 버그 바운티 프로그램을 진행해야 합니다
- 완전한 테스트를 진행할 때, 다양한 참가자들을 참가시켜 컨트랙트가 규모있게 실행될 수 있도록 해야 합니다
- 메인넷에 베타를 통해 배포하고 리스크 범위를 제한해야 합니다

##### 자동 폐기(Deprecation)

테스트를 진행하면서 추가적인 액션이 실행되지 않도록 일정 시간 이후 자동으로 폐기되는 방법을 쓸 수 있습니다. 예를 들면, 알파 수준의 컨트랙트가 몇 주간 동작하고 난 뒤 마지막에 출금을 제외한 모든 액션을 종료시키도록 할 수 있습니다.

```sol
modifier isActive() {
    require(block.number <= SOME_BLOCK_NUMBER);
    _;
}

function deposit() public isActive {
    // 코드
}

function withdraw() public {
    // 코드
}

```
##### 사용자 및 컨트랙트에 들어가는 이더의 양을 제한할 것

초기 상태에서는 사용자나 전체 컨트랙트에 대한 이더의 양을 제한하여 리스크를 줄일 수 있습니다.

### 버그 바운티 프로그램

바운티 프로그램을 운영하기 위한 팁입니다:

- 어떤 통화를 포상으로 줄 것인지 결정해야 합니다 (BTC 혹은 ETH 등)
- 포상금 총 예산의 추정를 결정해야 합니다
- 예산에 따라 3 단계로 보상을 정합니다
  - 보상하고 싶은 가장 적은 양의 수준
  - 보상하고 싶은 가장 높은 양의 수준
  - 심각한 취약점을 발견했을 때에 대해 보상을 받을 수 있는 추가 범위
- 바운티 심사위원이 누가 될 것인지 결정해야 합니다. (보통 3명이 적절할 겁니다)
- 리드 개발자가 바운티 심사위원 중의 한 명이어야 합니다
- 버그 리포트를 받았을 때, 리드 개발자는 심사위원진의 자문에 따라 버그의 심각성에 대해 평가해야 합니다
- 이 단계는 꼭 비공개 계정에서 진행하고 이슈를 깃허브에 정리합니다
- 버그가 반드시 수정되어야만 하는 종류라면 개발자는 버그를 확실하게 발생시켜 확인할 수 있는 테스트 케이스를 비공개 저장소에 작성합니다
- 개발자는 수정사항을 구현하고 테스트를 통과하는지 확인합니다. 필요시 추가적인 테스트 케이스를 작성할 수도 있습니다
- 바운티 사냥꾼에게 수정사항을 보여줍니다. 그리고 수정사항을 공개 저장소에 통합합니다
- 바운티 사냥꾼이 이 수정사항에 아무런 피드백이 없는지 확인합니다
- 바운티 심사위원은 해당 버그의 *발생 가능성* 과 *중요성* 에 의거하여 보상의 크기를 결정합니다
- 바운티 참가자들이 전체 프로세스 진행을 계속해서 알 수 있도록 하고, 보상의 지급이 늦어지지 않는 것을 크게 신경써야 합니다

예를 들어 다음 3 단계의 보상이 있습니다. [이더리움의 바운티 프로그램](https://bounty.ethereum.org)을 확인하세요:

> 지불되는 보상의 크기는 버그의 심각성의 크기에 따라 달라집니다. 마이너한 크게 무해한 버그의 경우에는 0.05 BTC에서 시작합니다. 중요한 버그의 경우에는 특히 합의 이슈 등과 관련한 경우 5BTC을 보상으로 받게 됩니다. 아주 심각한 취약점을 발견했을 경우에는 25 BTC까지 더욱 높은 보상 또한 가능합니다.
