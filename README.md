# Halmos vs Damn Vulnerable DeFi

이 문서는 [igorganich](https://github.com/igorganich)님께서 작성하신 [damn-vulnerable-defi-halmos](https://github.com/igorganich/damn-vulnerable-defi-halmos) 레포지토리를 gemini3.0으로 번역한 문서입니다.

Halmos vs Damn Vulnerable DeFi는 Halmos 심볼릭 분석기를 사용하여 Damn Vulnerable DeFi CTF를 해결하는 방법에 관한 기사 시리즈입니다.

theredguild가 완전히 foundry 기반 테스트로 마이그레이션된 Damn Vulnerable Defi v4를 출시했기 때문에, 이러한 문제들을 해결하는 데 Halmos를 사용하는 것이 매우 편리해졌습니다.

이 자료는 CTF에 대해 Halmos를 사용하거나 스마트 계약에 대한 공격을 구축하는 가능성에 대한 튜토리얼로 간주되어야 합니다.

Damn Vulnerable DeFi 솔루션의 맥락에서 Foundry fuzzing과 Echidna를 사용한 퍼징 기술과의 비교도 설명되었습니다.

**경고**: 이 저장소는 아직 진행 중이므로 모든 문제에 대한 솔루션이 없을 수 있습니다.

# 목표
이 작업의 결과로 다음 목표를 달성할 것으로 기대합니다:
1. Halmos에서 심볼릭 실행 테스트를 구축하고, 심볼릭 테스트를 위해 대상 계약을 조정하는 방법을 배웁니다.
2. Halmos 사용의 한계를 파악하고, 휴리스틱 및 최적화를 사용하여 이를 우회하는 방법을 찾습니다.
3. 버그를 찾고 복잡한 CTF 문제를 실제로 해결하는 기술을 축적합니다.
4. Halmos를 퍼징 엔진과 비교하여 일종의 "벤치마크"를 만듭니다.

# 관심 대상
1. 감사자, EVM 보안 전문가
2. CTF 참여자
3. Solidity 개발자
4. Solidity와 스마트 계약 보안을 배우는 초보자
5. 암호화폐 애호가


# 전제 조건
1. 독자는 Halmos 기초에 익숙해야 합니다: https://github.com/a16z/halmos
2. 이 자료에서는 Damn Vulnerable Defi 문제에 대한 설명이 아니라 Halmos 사용에 구체적으로 초점을 맞춥니다. 따라서 독자가 Damn Vulnerable DeFi에 익숙하고 이러한 문제를 해결하는 방법을 알고 있다고 가정합니다. 솔루션이 포함된 훌륭한 자료는 다음과 같습니다: https://www.youtube.com/playlist?list=PLwHGiYB583YuDoAjKPDfYMKOmuFIGJCnW

# 읽는 방법
새로운 자료, 새로운 기술 등의 프레젠테이션 편의를 위해 기사의 순서는 문제의 원래 순서와 다릅니다. 실제 순서:
1. [Unstoppable](https://github.com/Burnnnnny/damn-vulnerable-defi-halmos/tree/master/test/unstoppable)
2. [Truster](https://github.com/Burnnnnny/damn-vulnerable-defi-halmos/tree/master/test/truster)
3. [Naive-receiver](https://github.com/Burnnnnny/damn-vulnerable-defi-halmos/tree/master/test/naive-receiver)
4. [Side-entrance](https://github.com/Burnnnnny/damn-vulnerable-defi-halmos/tree/master/test/side-entrance)
5. [The-rewarder](https://github.com/Burnnnnny/damn-vulnerable-defi-halmos/tree/master/test/the-rewarder)
6. [Selfie](https://github.com/Burnnnnny/damn-vulnerable-defi-halmos/tree/master/test/selfie)
7. [Backdoor](https://github.com/Burnnnnny/damn-vulnerable-defi-halmos/tree/master/test/backdoor)
8. [Climber](https://github.com/Burnnnnny/damn-vulnerable-defi-halmos/tree/master/test/climber)
9. 미정 (TBD)
10. ...

# 기여
개선 아이디어가 있으면 주저하지 말고 풀 리퀘스트를 보내주세요.

또한 텔레그램으로 저에게 연락할 수 있습니다: https://t.me/super_uzer

지원을 해주신 Halmos 개발팀에게 특별한 감사를 드립니다 :)
