---
description: Unity + Photon Quantum 3를 선택한 배경에 대해서 다룹니다.
icon: '0'
---

# 기술 선택 배경

## Reference

{% embed url="https://doc.photonengine.com/quantum/current/quantum-intro" %}

## 게임 엔진 선택

모바일 3D 멀티플레이어 게임 구현을 위한 게임 엔진으로는 유니티를 선택했다. 유니티는 다음과 같은 장점이 있기에 선택했다.

* 직관적인 에디터 UI
* 안전한 메모리 관리를 지향하고, 문법이 C++에 비해 간결하므로 프로그래밍 측면에서 생산성이 높음
* 유니티로 여러 프로젝트를 진행해봤기에 익숙함

## 서버의 구현과 성능 최적화

게임 엔진으로 유니티를 선택했다면, 다음으로 고려해야 할 것은 '서버의 구현’이고 그 다음으로는 ‘성능 최적화‘를 고려해야 한다. 서버의 구현은 물론이고 클라이언트의 성능을 최적화하여 기기 사양에 관계 없이 다양한 기기에서 플레이할 수 있어야 하며, 네트워크로 전송될 데이터를 최소화하여 트래픽에 소모될 비용 또한 줄여야 한다.

모바일 실시간 멀티플레이를 위한 서버를 직접 구현해도 되지만, 최적의 서버를 구현하기에는 많은 시행착오와 테스트를 거쳐야 한다. 서버 구현에 있어서 중요한 점을 나열하면 다음과 같다.

* Scalability(Load Balancing) : 플레이어의 수가 급증할 경우와 급감할 경우에 대해서 게임 서버를 어떻게 관리할 것인가? 플레이어의 수가 많아져도 서버는 이를 감당할 수 있는지
* Latency for Various Regions : 다양한 지역의 사람들을 어떻게 연결할 것인지
* Cost : 클라우드 네트워크 사용 비용은 최적화가 되어 있는지, 불필요한 데이터를 전송하고 있지는 않은지
* Cheat Protection : 특정 플레이어가 비정상적인 game state를 서버에 보낸다면?
* Session Management : 매칭이 성공하면 매칭이 성공한 플레이어들의 세션을 연결이 끊길 때까지 수많은 세션 룸에서 유지해야 함

게임 클라이언트 개발에 집중하기 위해서는 위 고려사항들을 포함하는 서비스를 사용해야 한다. 따라서 그 서비스는 Unity와 통합된 SDK이어야 하고, 클라우드 서비스까지 지원해야 한다. 이를 위한 서비스로 대표적인 것은 3개가 있었다.

* Photon Quantum 3
* Unity Gaming Service
* AWS Gamelift

Unity Gaming Service는 Unity DOTS와 결합해서 사용하면 좋은 성능을 낼 수 있다고 조사되었으나, 서버-클라이언트 테스팅 방식이 파일 업로드 방식으로 이루어져 잦은 코드 변경에 적합하지 않을 것 같아 배제했다.

AWS Gamelift는 매치메이킹과 같은 멀티플레이어 서비스를 지원하지만, 클라우드 서비스에 집중하고 클라이언트의 성능 최적화에는 관여하지 않는다.

## Unity + Photon Quantum의 구조

따라서 Act like AI 개발에는 멀티플레이어를 지원하고 High Performance를 보여줄 수 있는 Photon Quantum 3를 선택했다.

<figure><img src="../.gitbook/assets/quantum-structure (1).png" alt=""><figcaption><p>Photon Quantum3의 구조</p></figcaption></figure>

위 구조는 Unity + Photon Quantum3를 사용할 때의 구조이다. 상위 계층부터 보면, 전반적인 구조는 Quantum API를 사용하여 ECS구조로 Simulation을 작성하고, Simulation에서 발생하는 이벤트를 View에서 Subscribe하는 방식으로 이루어진다.

Simulation 레벨에서는 View 레벨의 유니티를 알지 못하므로, Simulation 레벨에서는 Pure C#과 Qunatum API(Data Structure, Physics 등)만 사용 가능하다.

네트워크를 통해 전달될 핵심 데이터인 Input은 계층 구조에서, 로컬 플레이어의 Input은 View → Simulation → Input → Network로 전달되고, 다른 플레이어의 Input은 Network → Input → Simulation → View 순서로 전달된다.

네트워크 계층에서는 릴레이 서버로 플레이어 접속 시 상태 스냅샷과 Input을 전송한다.

## Photon Quantum 3가 갖는 특성

Photon Quantum 3가 갖는 특성은 다음과 같다.

* 클라우드 기반 멀티플레이어 지원
  * 매치 메이킹 가능
  * 파티 플레이 가능
* Cost Efficient
  * 복잡한 컨테이너 오케스트레이션을 사용하지 않고, 릴레이 서버 형식으로 서버를 구현했기에 서버 운영 및 네트워크 트래픽 비용을 고려했을 때 Efficient함
    * 서버의 자세한 구현은 Quantum 서버의 내부 기술이므로 알 수 없음
  * 네트워크로 전송할, 직렬화할 데이터를 최적화하는 것을 지원해 네트워크 트래픽을 최소화할 수 있게 함
*   ECS

    * ECS 패턴을 기반으로 유니티와 통합된 고성능 프레임워크를 구현함. ECS의 자세한 원리는 후술

    > _The key to Quantum's high performance is the use of pointer-based C# code combined with its **sparse-set ECS memory model** (all based **memory aligned data-structs** and a **custom heap allocator** - **no garbage collection at runtime from simulation code**)._

    > _In Quantum, all gameplay data (game state) is kept either in the **sparse-set ECS data structures** (entities and components) or in our custom heap-allocator (dynamic collections and custom data), always as **blittable memory-aligned C# structs.**_

    * Blittable 타입은 추가적인 데이터 변환 없이 바로 **메모리 복사**가 가능한 타입을 말함
      * C# 구조체를 Unmanaged Code(포인터)로 넘기거나, 반대로 Unmanaged Memory에서 C# 구조체로 읽을 때, 특별한 변환 과정 없이 바로 메모리에서 데이터를 복사할 수 있음
      * Blittable 타입은 포인터 연산이 가능하여, 마샬링(marshaling, Managed Code와 Unmanaged Code 간에 데이터를 변환하고 전송하는 과정)이 필요 없음
      * `int`, `float`, `double`, `byte` 는 Blittable Type
      * `string`, `object` 는 Blittable이 아님
    * Custom Heap Allocator를 통한 런타임 가비지 컬렉션 회피
    * Sparse Set, Blittable 타입을 활용하여 연속적으로 메모리에 데이터를 저장하고 캐시 지역성 극대화
* Deterministic in Cross-Platform
  * Photon Quantum 3는 Cross-Platform을 지원하는, 결정론적으로 게임 로직을 실행하는 게임 엔진임
    * 결정론적으로 동작한다는 뜻은 각 클라이언트에서 독립적으로 시뮬레이션을 실행하는데 이 때 동일한 입력값에 대해 항상 동일하게 동작한다는 뜻
  * Simulation이 ‘결정론적’으로 동작하기 때문에 독립된 클라이언트 간에 최소한의 데이터 교환만으로 플레이어 간 동기화가 가능함
  * 한 클라이언트의 상태 스냅샷을 다른 클라이언트로 전달한 이후, 클라이언트 간 교환할 데이터는 클라이언트 각각의 인풋 뿐
    * 클라이언트 간 교환하는 데이터가 인풋뿐이니 Cross-Platform에서 정상적으로 동작하려면 각 클라이언트에서 실행되는 시뮬레이션은 완벽히 일치해야 함
      * 이를 위해 Quantum은 커스텀 자료형을 만들어 둠
      * 동일한 입력에 동일한 동작을 멀티플랫폼에서 구현하기 위해 Quantum은 부동 소수점 대신 고정 소수점 자료형만을 사용함. 부동 소수점의 오차는 허용할 수 없기 때문
      * 랜덤값 또한 클라이언트 간 동일해야 하므로 특수 자료형을 사용함
* Without Lockstep
  * Lockstep Approach에서 모든 클라이언트는 한 틱이 끝날 때까지 기다려야 함
  * Soft Lockstep은 이 틱을 기다릴 필요가 없게 함
    * Quantum에서 로컬 시뮬레이션을 예측하여 실행하고, 서버에서 롤백 시스템을 구현하여 예측이 잘못되었을 경우 re-simulate하기 때문
  * 물론 Quantum은 Lockstep을 사용할 수 있는 옵션이 있지만 권장되진 않음
* Input의 동기화
  * Position 동기화 방식이 아니라 Input 동기화 방식을 사용함
    * Input을 View와 독립된 Simulation에서 조작하여 플레이어처럼 행동하는 봇의 움직임 생성, 게임 리플레이, 플레이어와 봇을 포함한 실제 게임 플레이 테스트 시나리오 같은 것을 제작하기가 훨씬 용이함
* Photon Cloud를 사용한 많은 프로젝트들
  * Over 1.4 billion monthly players in global Photon Cloud (Quantum을 포함한 모든 Photon 서비스에 대한 월간 사용자 수)
  * 클라우드 규모가 꽤 있는 편이기에, 갑자기 클라우드 서비스가 중단될 위험은 적다고 판단함
