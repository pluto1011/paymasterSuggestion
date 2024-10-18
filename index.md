---
layout: default
---
# Paymaster 취약점 분석

Text can be **bold**, *italic*, ~~strikethrough~~ or `keyword`.

[Link to another page](./another-page.html).

There should be whitespace between paragraphs.

There should be whitespace between paragraphs. We recommend including a README, or a file with information about your project.

## 소개

이 문서는 Microsoft의 'DREAD' 위협 평가 프레임워크를 기반으로 Paymaster의 다양한 취약점을 분석합니다. 취약점은 high 등급이 많고, 관련 취약점이 여러 개일수록 우선순위가 높습니다.

> **DREAD 프레임워크:**
> - **D**amage potential (피해 정도): 공격이 성공했을 때 시스템에 가해지는 피해 정도
> - **R**eproducibility (재현 가능성): 공격이 얼마나 쉽게 재현될 수 있는지
> - **E**xploitability (악용 가능성): 공격을 실행하기 위해 필요한 노력의 정도
> - **A**ffected users (영향받는 사용자 수): 이 공격이 영향을 미치는 사용자 수
> - **D**iscoverability (발견 가능성): 공격이 얼마나 쉽게 발견될 수 있는지

## Paymaster 공통 취약점

### 1. 잘못된 nonce 관리/검증

- 중복 해시를 저장하여 보관하는 Paymaster 사례의 문제점:
  * 계정을 참조하는 nonce이기 때문에 replay가 불가능함
  * 중복된 값이 들어왔을 때 저장하여 걸러내면 문제 발생 가능
  * paymasterAndData에서 특정 값을 파라미터로 사용할 경우 해당 부분이 누락되면 문제 발생 가능

### 2. 잘못된 자금 관리

#### 2.1 자금 인출 로직의 부재

- **설명**: Paymaster 내부에 받은 토큰/ETH를 다시 뺄 수 있는 로직이 존재하지 않을 시, 사용자의 자금이 묶일 수 있음
- **완화 방법**: onlyOwner 또는 입금자에 맞춰서 자금을 빼는 로직을 추가
- **영향**: MEDIUM

#### 2.2 자금 변수 업데이트의 부재

- **설명**: postOp에서 돈을 환급받은 이후 deposit을 관리하는 mapping 변수를 초기화하지 않으면 postOp에서 환급받은 후 withdraw하여 돈을 이중으로 가져갈 수 있음
- **완화 방법**: postOp에서 돈을 환급받는 경우, 자금을 관리하는 mapping 변수를 업데이트
- **영향**: HIGH

> 참고: 오프체인에서 Paymaster의 자금을 모니터링하다가 채워주는 경우도 있음

### 3. 잘못된 검사 로직

#### 3.1 서명오류 시 잘못된 처리

- **설명**: Paymaster와 user account의 서명 검증 실패 시 revert를 하는 것으로 처리하는 구현체들이 존재
- **완화 방법**: 서명 검증 실패 시 revert가 아니라 `SIG_VALIDATION_FAILED`를 반환하는 것으로 처리
- **영향**: MEDIUM

#### 3.2 postOpGasLimit 범위 검증 부재

- **설명**: validate 단계에서 postOpGasLimit에 대한 검증이 없으면 악의적인 유저가 postOp에 대한 가스비를 0으로 설정하거나 낮게 설정했을 때 postOp에서 제대로 정산이 이루어지지 않은 채로, postOp가 종료됨
- **완화 방법**: postOpGasLimit의 최솟값을 확인하는 로직을 포함
- **영향**: HIGH

#### 3.3 signer 변경 시 일시적인 DoS

- **설명**: validation에서 paymaster가 서명을 하는 로직을 가지고 있을 때, signer를 변경하면 아직 처리되지 못하고 대기하던 UserOperation에 대해서 일시적으로 DoS가 발생하는 경우가 존재
- **완화 방법**: setSigner 함수 안에서 oldSigner의 값을 캐싱해서 일시적으로 사용 (결국 setSigner를 2번 호출해야 함)
- **영향**: LOW

### 4. 정확한 계산 관련

#### 4.1 할인금액 부담주체 고려 부족

- **설명**: priceMarkup을 사용할 때, 할인로직이 포함된다면 할인금액이 어디서 부담되는지 주의해야 함
- **완화 방법**: 금액의 흐름을 보면서 비즈니스 로직에 맞게 금액을 차감
- **영향**: HIGH

#### 4.2 가스비 계산 시, 잘못된 수수료 범위제한

- **설명**: priceMarkup을 포함한 지불금액에 직접적인 영향을 주는 변수들을 이용해 계산할 때 분자와 분모 비율에 주의해야 함
- **완화 방법**: 입력받는 값에 대해서 최소/최대 limit을 올바르게 정함
- **영향**: HIGH

#### 4.3 unchecked 구문 사용 시 overflow / underflow 가능성

- **설명**: 가스를 절약하기 위해 unchecked 구문 사용 시 민감한 부분이나, 유저의 파라미터를 이용하여 계산에서 오버/언더 플로우가 발생할 수 있는 경우에는 사용에 주의해야 함
- **완화 방법**: unchecked 구문 내에서 유저의 파라미터를 이용하거나, critical하게 동작할 수 있는 부분에 대해서는 추가적인 검증을 거침. 또는 unchecked 블록에서 제외
- **영향**: CRITICAL

### 5. 구조적 오류

#### 5.1 올바르지 않은 버전의 참조 및 구현

- **설명**: paymaster가 구현하고 있는 인터페이스는 v0.6 이지만 참조하는 entrypoint는 v0.7을 가리키는 경우가 존재
- **완화 방법**: 페이마스터를 v0.7에 맞춰서 구현하거나 entrypoint를 v0.6으로 구현하여 버전을 통일
- **영향**: HIGH

#### 5.2 validation 과정에서 올바르지 않은 형식의 UserOp에 대한 처리 부족

- **설명**: 해당 페이마스터의 형식을 지키지 않는 UserOperation이 들어왔을 경우에는 invalid한 형식임을 알리고 올바르게 revert 시키지 않아, 번들러의 validation 단계를 통과
- **완화 방법**: validate과정에서 파싱하면서 형식이 올바르지 않은 것은 revert 처리
- **영향**: LOW

### 6. 평판관련

#### 6.1 낮은 gas price로 인한 평판 공격

- **설명**: 유저가 낮은 gas price의 UserOperation을 멤풀에 제출하면, 번들러는 해당 UserOperation의 가치가 상대적으로 낮기때문에 번들에 포함시키지 않고 멤풀에 대기시킴. 그렇게 되면 opsSeen만 증가하고 opsInclude는 증가하지 않기때문에 관련 엔티티들의 명성에 영향이 끼쳐짐
- **완화 방법 아이디어**: mempool에 userOp이 일정 시간 이상 있다 -> user가 gas를 너무 적게 줬으니 user 탓이다 -> userOp을 mempool에서 걸러내고 paymaster의 opseen 1 낮추고 user 평판 깎는다
- **고려할 점**: 받아들일 수 있는 gasPrice의 최솟값을 설정하여 멤풀에 번들로부터 수용될 수 없는 userOperation을 걸러서 받는다

### 7. 서명 재사용

#### 7.1 Cross-chain 서명 재사용 공격 가능성

- **설명**: 페이마스터의 validatePaymasterUserOp에서 파라미터로 전달된 userOpHash를 사용하지 않을 때, 페이마스터 내부의 getHash 함수에 chainId를 포함시키지 않으면 다른 체인에서 페이마스터 서명 재사용이 가능
- **완화 방법**: getHash 함수에 chainId를 포함
- **영향**: MEDIUM

#### 7.2 nonce를 포함하지 않는 서명

- **설명**: VerifyingPaymaster와 같이 검증 과정에서 Paymaster의 signature가 필요한 경우 `getHash`함수에 userOp의 nonce가 제외되어있는 경우, replay공격에 취약
- **완화 방법**: 서명할 때 nonce를 포함하여 계산
- **영향**: HIGH

## Sponsorship Paymaster 한정 취약점

### 1. 트랜잭션의 가스 제한 로직 부재

- **설명**: validation phase에서 txAmount<userLimit일때만 허용하는 경우, 각 tx는 임의의 gasLimit과 maxFeePerGas를 가질 수 있으므로 gas가 많이 소모되는 tx를 보내면 paymaster의 balance를 고갈시켜 다른 유저들이 피해 볼 수 있음
- **완화 방법**: 각 유저에게 특정 개수의 tx가 아닌, 특정 양의 gas를 할당. 또는, 각 tx에 대해 gasLimit과 maxFeePerGas에 대한 상한선을 지정
- **영향**: MEDIUM

### 2. 유저 부담 비용 부재

- **설명**: UserOp 실행에서 유저가 부담하는 비용이 0이면 유저는 많은 양의 spam UserOp를 전송하여 sponsorship paymaster의 자금을 고갈시킬 수 있음
- **완화 방법**: 공격 비용이 0을 초과하게끔 userOp.sender에게 약간의 비용을 부담하게 함
- **영향**: MEDIUM

```javascript
// 코드 예시
function validatePaymasterUserOp(UserOperation calldata userOp, bytes32 userOpHash, uint256 maxCost)
    external override returns (bytes memory context, uint256 validationData) {
    // 유저에게 최소한의 비용 부담
    require(userOp.verificationGasLimit > MIN_VERIFICATION_GAS, "Insufficient verification gas");
    // ... 나머지 검증 로직
}
```

* * *

## 추가 고려사항

- 유저가 uint max로 approve 했을 때, 페이마스터가 러그풀 할 수 있는 케이스 검토
- 페이마스터에서 토큰의 decimal을 제대로 반영 안하는 케이스 분석
- 유저의 의도대로 maxFee가 제대로 반영되는지 확인

* * *

이 문서는 Paymaster의 주요 취약점과 그에 대한 완화 방법을 제시합니다. 실제 구현 시 이러한 취약점들을 고려하여 보안을 강화해야 합니다.
