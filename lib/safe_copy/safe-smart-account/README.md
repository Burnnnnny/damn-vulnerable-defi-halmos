Safe Contracts
==============

[![npm version](https://badge.fury.io/js/%40gnosis.pm%2Fsafe-contracts.svg)](https://badge.fury.io/js/%40gnosis.pm%2Fsafe-contracts)
[![Build Status](https://github.com/safe-global/safe-contracts/workflows/safe-contracts/badge.svg?branch=development)](https://github.com/safe-global/safe-contracts/actions)
[![Coverage Status](https://coveralls.io/repos/github/safe-global/safe-contracts/badge.svg?branch=development)](https://coveralls.io/github/safe-global/safe-contracts)

> :warning: **이 브랜치는 개발 중인 변경 사항을 포함하고 있습니다.** 최신 감사를 받은 버전을 사용하려면 올바른 커밋을 사용해야 합니다. Safe 팀이 사용하는 태그된 버전은 [releases](https://github.com/safe-global/safe-contracts/releases)에서 찾을 수 있습니다.

사용법
-----
### yarn으로 요구 사항 설치:

```bash
yarn
```

### 테스트

테스트 실행 방법:

```bash
yarn build
yarn test
```

선택적으로, ERC-4337 호환성 테스트를 실행하려면 실시간 번들러와 노드를 사용하므로 몇 가지 전제 조건이 필요합니다:

1. 환경 변수 정의:

```
ERC4337_TEST_BUNDLER_URL=
ERC4337_TEST_NODE_URL=
ERC4337_TEST_SINGLETON_ADDRESS=
ERC4337_TEST_SAFE_FACTORY_ADDRESS=
MNEMONIC=
```

2. 니모닉에서 파생된 실행자 계정에 ERC4337 모듈 배포와 테스트 작업을 위한 Safe의 사전 자금 조달을 충당할 수 있도록 Native Token을 미리 충전합니다.

### 배포

다양한 Safe 계약 배포 및 해당 주소 모음은 [Safe deployments](https://github.com/safe-global/safe-deployments) 저장소에서 찾을 수 있습니다.

새 네트워크에 대한 지원을 추가하려면 ``Deploy`` 섹션의 단계를 따르고 [Safe deployments](https://github.com/safe-global/safe-deployments) 저장소에 PR을 생성하세요.

### 배포 (Deploy)

> :warning: **계약을 배포할 때 올바른 커밋을 사용해야 합니다.** 계약 파일 내의 변경 사항(주석 포함)은 다른 주소를 초래합니다. Safe 팀이 사용하는 태그된 버전은 [releases](https://github.com/safe-global/safe-contracts/releases)에서 찾을 수 있습니다.

> **현재 버전:** 최신 릴리스는 [v1.3.0-libs.0](https://github.com/safe-global/safe-contracts/tree/v1.3.0-libs.0)이며 커밋은 [767ef36](https://github.com/safe-global/safe-contracts/commit/767ef36bba88bdbc0c9fe3708a4290cabef4c376)입니다.

이것은 결정론적으로 계약을 배포하고 기본적으로 [Solidity 0.7.6](https://github.com/ethereum/solidity/releases/tag/v0.7.6)을 사용하여 etherscan에서 계약을 확인합니다.

준비:
- `.env`에 `MNEMONIC` 설정
- `.env`에 `INFURA_KEY` 설정

```bash
yarn deploy-all <network>
```

이것은 다음 단계를 수행합니다:

```bash
yarn build
yarn hardhat --network <network> deploy
yarn hardhat --network <network> sourcify
yarn hardhat --network <network> etherscan-verify
yarn hardhat --network <network> local-verify
```

#### 사용자 정의 네트워크

`NODE_URL` 환경 변수를 사용하여 RPC 엔드포인트를 통해 모든 EVM 기반 네트워크에 연결할 수 있습니다. 이 연결은 `custom` 네트워크와 함께 사용할 수 있습니다.

예: 해당 네트워크에 Safe 계약 제품군을 배포하려면 `yarn deploy-all custom`을 실행합니다.

결과 주소는 모든 네트워크에서 동일해야 합니다.

참고: 계약 코드가 변경되거나 다른 Solidity 버전이 사용되면 주소가 달라집니다.

#### 리플레이 보호 (EIP-155)

일부 네트워크는 리플레이 보호를 요구하므로 리플레이 보호 없이 사전 서명된 트랜잭션에 의존하는 기본 배포 프로세스와 호환되지 않습니다 (참조: https://github.com/Arachnid/deterministic-deployment-proxy).

Safe 계약은 다른 결정론적 배포 프록시(https://github.com/safe-global/safe-singleton-factory)를 사용합니다. 이 패키지의 최신 버전이 설치되었는지 확인하려면 배포 전에 `yarn add @gnosis.pm/safe-singleton-factory`를 실행하세요. 팩토리를 새 네트워크에 배포하는 방법을 포함한 자세한 내용은 팩토리 저장소를 참조하세요.

참고: 이것은 hardhat의 기본 결정론적 배포 프로세스와 비교하여 다른 주소를 초래합니다.

### 계약 확인

이 명령은 배포 아티팩트를 사용하여 계약을 컴파일하고 온체인 코드와 비교합니다.
```bash
yarn hardhat --network <network> local-verify
```

이 명령은 계약 소스를 Etherescan에 업로드합니다.
```bash
yarn hardhat --network <network> etherscan-verify
```

문서
-------------
- [Safe 개발자 포털](http://docs.safe.global)
- [에러 코드](docs/error_codes.md)
- [코딩 가이드라인](docs/guidelines.md)

감사/ 형식 검증
---------
- [버전 1.4.0/1.4.1 (Ackee Blockchain)](docs/audit_1_4_0.md)
- [버전 1.3.0 (G0 Group)](docs/audit_1_3_0.md)
- [버전 1.2.0 (G0 Group)](docs/audit_1_2_0.md)
- [버전 1.1.1 (G0 Group)](docs/audit_1_1_1.md)
- [버전 1.0.0 (Runtime Verification)](docs/rv_1_0_0.md)
- [버전 0.0.1 (Alexey Akhunov)](docs/alexey_audit.md)

보안 및 책임
----------------------
모든 계약은 어떠한 보증도 없이 제공되며, 상품성 또는 특정 목적에의 적합성에 대한 묵시적 보증조차 없습니다.

라이선스
-------
모든 스마트 계약은 LGPL-3.0에 따라 배포됩니다.
