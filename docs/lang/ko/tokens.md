
# 토큰 구현 모범 사례

토큰을 구현할 때 일반적인 모범 사례들 또한 준수해야하지만, 특별히 고려해야하는 사항들이 따로 있습니다

## 최신 기준을 준수하세요

일반적으로 말하자면, 토큰 스마트 컨트랙트는 수용가능하고 안정적인 기준을 따라야 합니다.

현재 통용되는 기준들은 다음과 같습니다:

* [EIP20](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-20.md)
* [EIP721](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-721.md) (대체 불가능 토큰)

## EIP-20은 선행 매매 공격이 가능한 것을 염두에 두세요

EIP-20 토큰의 `approve()` 함수는 승인된 지출자가 실제 의도 이상의 금액을 사용할 수 있도록 하는 가능성을 내포하고 있습니다. 승인된 지출자가 `approve()`가 처리되기 직전 및 직후에 `transferFrom()` 함수를 호출할 수 있도록 하여 [선행 매매 공격](./known_attacks/#transaction-ordering-dependence-tod-front-running)을 할 수 있습니다. 더 자세한 내용은 [EIP](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-20.md#approve)와 [이 문서](https://docs.google.com/document/d/1YLPtQxZu1UAvO9cZ1O2RPXBbT0mooh4DYKjA_jp-RLM/edit)를 확인해주세요.


## 토큰을 0x0 주소로 송금하는 것을 방지하세요

글의 작성 시점 기준으로는, "영주소(zero address, [0x0000000000000000000000000000000000000000](https://etherscan.io/address/0x0000000000000000000000000000000000000000))"에 80$ million 가치에 해당하는 토큰이 묶여 있습니다.

## 자기 컨트랙트 주소에 토큰을 송금하는 것을 방지하세요

토큰을 자기 컨트랙트 주소에 송금하는 것을 방지하세요

이 것을 그대로 두어 토큰을 잃어버리는 가능성을 보여준 예시가 [EOS의 토큰 스마트 컨트랙트](https://etherscan.io/address/0x86fa049857e0209aa7d9e616f7eb3b3b78ecfdb0)입니다. 여기에는 90,000개의 토큰이 해당 컨트랙트 주소에 묶여있습니다.

### 예시

추천사항을 만족하는 구현 예시는 다음과 같이 "to" 주소가 0x0도 아니고 스마트 컨트랙트 고유의 주소도 아닌 것을 확인하는 한정자를 구현하는 것입니다:

```sol
    modifier validDestination( address to ) {
        require(to != address(0x0));
        require(to != address(this) );
        _;
    }
```

한정자는 반드시 `transfer`와 `transferFrom` 메소드에 적용되어야 합니다:

```sol
    function transfer(address _to, uint _value)
        validDestination(_to)
        returns (bool)
    {
        (... 로직 ...)
    }

    function transferFrom(address _from, address _to, uint _value)
        validDestination(_to)
        returns (bool)
    {
        (... 로직 ...)
    }
```
