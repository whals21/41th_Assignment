# Product Requirements Document (PRD)
## 3D 액션 게임 - 적 AI 및 전투 시스템

### 프로젝트 개요
Unity 기반 3D 액션 게임으로, 적 AI의 상태 패턴 기반 행동 시스템과 근접 전투 메커니즘을 구현하는 프로젝트입니다.

---

## 1. 프로젝트 목표

### 1.1 핵심 목표
- 상태 패턴 기반의 적 AI 시스템 구현
- 플레이어와 적 간의 근접 전투 시스템 구현
- 거리 기반 적 행동 변화 시스템 구현
- 기본적인 플레이어 액션 시스템 구현

### 1.2 학습 목표
- Unity에서 상태 패턴(State Pattern) 구현 경험
- 3D 공간에서의 거리 계산 및 영역 판정
- 충돌 감지 및 피격 판정 시스템 구현
- 기본적인 게임 AI 로직 설계

---

## 2. 기능 요구사항

### 2.1 적 AI 시스템

#### 2.1.1 상태 패턴
적은 다음 3가지 상태를 가지며, 상태 전환은 플레이어와의 거리에 따라 결정됩니다.

**상태 1: 배회 (Patrol)**
- 설명: 기본 상태로, 정해진 경로나 영역을 배회
- 조건: 플레이어가 추적 영역 밖에 있을 때
- 행동:
  - 지정된 웨이포인트를 순환하며 이동
  - 또는 랜덤한 위치로 이동 후 대기
  - 느린 이동 속도

**상태 2: 추적 (Chase)**
- 설명: 플레이어를 발견하고 추적하는 상태
- 조건: 플레이어가 추적 영역 안에 진입했을 때
- 행동:
  - NavMesh를 이용한 플레이어 추적
  - 빠른 이동 속도
  - 플레이어와의 거리 유지 시도
- 전환:
  - 공격 영역 진입 시 → 공격 상태
  - 추적 영역 이탈 시 → 배회 상태

**상태 3: 공격 (Attack)**
- 설명: 플레이어를 공격하는 상태
- 조건: 플레이어가 공격 영역 안에 진입했을 때
- 행동:
  - 공격 애니메이션 재생
  - 공격 판정 활성화
  - 공격 후 쿨타임 적용
- 전환:
  - 공격 영역 이탈 시 → 추적 상태

#### 2.1.2 영역 설정
- **추적 영역 (Chase Range)**: 반경 10m (조정 가능)
- **공격 영역 (Attack Range)**: 반경 2m (조정 가능)
- Scene View에서 Gizmos로 영역 시각화

#### 2.1.3 기술 구현
- State Pattern 사용
- NavMesh를 이용한 경로 탐색
- 거리 계산: `Vector3.Distance()`
- 상태 전환 로직: Update 또는 Coroutine

### 2.2 근접 전투 시스템

#### 2.2.1 공격 판정 시스템
**피격 판정 영역**
- 무기(검)에 Collider 추가
- 공격 애니메이션 중 특정 프레임에만 Collider 활성화
- Animation Event 또는 Animator 상태 기반 제어

**충돌 감지**
- Trigger Collider 사용
- OnTriggerEnter로 피격 대상 감지
- Layer 또는 Tag를 이용한 타겟 필터링

**데미지 처리**
- 데미지 값 설정 (기본: 10)
- 피격 대상의 체력 감소
- 피격 피드백 (애니메이션, 이펙트, 사운드)

#### 2.2.2 적 근접 공격
- 공격 애니메이션 재생
- 공격 판정 영역 활성화 (애니메이션 특정 구간)
- 플레이어 피격 시 데미지 전달

#### 2.2.3 플레이어 근접 공격
- 공격 버튼 입력 (마우스 좌클릭 또는 키보드)
- 공격 애니메이션 재생
- 검에 부착된 Collider로 피격 판정
- 적 피격 시 데미지 전달

### 2.3 플레이어 시스템

#### 2.3.1 필수 기능
**이동 (Movement)**
- WASD 또는 방향키 입력
- CharacterController 또는 Rigidbody 사용
- 이동 속도: 5m/s (조정 가능)
- 카메라 방향 기준 이동

**점프 (Jump)**
- 스페이스바 입력
- 중력 적용
- 이중 점프 방지 (지면 체크)

**공격 (Attack)**
- 마우스 좌클릭 또는 특정 키 입력
- 공격 애니메이션 재생
- 공격 중 이동 제한 (선택 사항)
- 공격 쿨타임 적용

#### 2.3.2 고급 기능 (선택 사항)
**패링 (Parry)**
- 특정 키 입력 (마우스 우클릭 또는 Shift)
- 패링 애니메이션 재생
- 패링 타이밍 윈도우 (0.3~0.5초)
- 성공 시:
  - 적 공격 무효화
  - 적 경직 상태 진입
  - 반격 가능 상태
- 실패 시:
  - 패링 쿨타임 적용
  - 일반 피격

---

## 3. 기술 사양

### 3.1 개발 환경
- **엔진**: Unity (2021.3 LTS 이상 권장)
- **언어**: C#
- **버전 관리**: Git

### 3.2 핵심 컴포넌트

#### 3.2.1 적 AI 관련
```
EnemyAI.cs - 메인 AI 컨트롤러
├─ IEnemyState.cs (인터페이스)
├─ PatrolState.cs
├─ ChaseState.cs
└─ AttackState.cs
```

#### 3.2.2 플레이어 관련
```
PlayerController.cs - 이동, 점프 제어
PlayerCombat.cs - 공격, 패링 제어
PlayerHealth.cs - 체력 관리
```

#### 3.2.3 전투 관련
```
DamageDealer.cs - 데미지 전달 (무기에 부착)
HealthSystem.cs - 체력 시스템 (플레이어/적 공통)
HitDetector.cs - 피격 감지
```

### 3.3 Unity 기능 활용
- **NavMesh**: 적 AI 경로 탐색
- **Animation**: Animator Controller, Animation Event
- **Physics**: Collider, Trigger, Layer
- **Input System**: 구 Input 또는 새 Input System

---

## 4. 시스템 플로우

### 4.1 적 AI 행동 플로우
```
[시작]
  ↓
[배회 상태]
  ↓ (플레이어 추적 영역 진입)
[추적 상태]
  ↓ (플레이어 공격 영역 진입)
[공격 상태]
  ↓ (공격 실행)
[쿨타임]
  ↓
[거리 재확인] → 상태 재평가
```

### 4.2 전투 플로우
```
[공격 입력]
  ↓
[공격 애니메이션 시작]
  ↓
[Animation Event: 피격 판정 활성화]
  ↓
[Collider Trigger 감지]
  ↓
[데미지 계산 및 전달]
  ↓
[피격 대상 체력 감소]
  ↓
[Animation Event: 피격 판정 비활성화]
  ↓
[공격 애니메이션 종료]
```

### 4.3 패링 플로우
```
[패링 입력]
  ↓
[패링 윈도우 시작 (0.3초)]
  ↓
  ├─ [적 공격 감지] → [패링 성공] → [적 경직]
  └─ [시간 경과] → [패링 실패] → [쿨타임]
```

---

## 5. 데이터 구조

### 5.1 적 설정 데이터
```csharp
[System.Serializable]
public class EnemySettings
{
    public float patrolSpeed = 2f;
    public float chaseSpeed = 5f;
    public float chaseRange = 10f;
    public float attackRange = 2f;
    public float attackCooldown = 2f;
    public int attackDamage = 10;
    public int maxHealth = 100;
}
```

### 5.2 플레이어 설정 데이터
```csharp
[System.Serializable]
public class PlayerSettings
{
    public float moveSpeed = 5f;
    public float jumpForce = 5f;
    public float attackCooldown = 1f;
    public int attackDamage = 20;
    public int maxHealth = 100;

    // 패링 관련
    public float parryWindow = 0.3f;
    public float parryCooldown = 1f;
}
```

---

## 6. 구현 우선순위

### Phase 1: 기본 플레이어 시스템 (1일)
- [ ] 플레이어 이동 구현
- [ ] 플레이어 점프 구현
- [ ] 기본 카메라 설정

### Phase 2: 플레이어 전투 시스템 (1일)
- [ ] 공격 입력 처리
- [ ] 공격 애니메이션 연동
- [ ] 검 피격 판정 구현
- [ ] 데미지 시스템 구현

### Phase 3: 적 AI 기본 구조 (1일)
- [ ] 상태 패턴 구조 설계
- [ ] 배회 상태 구현
- [ ] 추적 상태 구현
- [ ] NavMesh 설정

### Phase 4: 적 전투 시스템 (1일)
- [ ] 공격 상태 구현
- [ ] 적 공격 판정 구현
- [ ] 상태 전환 로직 완성

### Phase 5: 고급 기능 (선택, 1-2일)
- [ ] 패링 시스템 구현
- [ ] 이펙트 및 사운드 추가
- [ ] UI (체력바 등) 구현

---

## 7. 테스트 계획

### 7.1 기능 테스트
- [ ] 적 배회 동작 확인
- [ ] 플레이어 접근 시 추적 확인
- [ ] 공격 범위 진입 시 공격 확인
- [ ] 플레이어 공격 판정 확인
- [ ] 적 공격 판정 확인
- [ ] 데미지 처리 확인
- [ ] 패링 성공/실패 확인

### 7.2 엣지 케이스
- [ ] 여러 적 동시 처리
- [ ] 빠른 입력 처리 (공격 스팸)
- [ ] 영역 경계에서의 상태 전환
- [ ] 장애물이 있을 때 NavMesh 동작

---

## 8. 성공 지표

### 8.1 필수 달성 목표
- 적 AI가 3가지 상태를 정상적으로 전환
- 플레이어와 적의 근접 공격이 정확히 작동
- 피격 판정이 정확하게 동작
- 플레이어 기본 조작 (이동, 점프, 공격) 완성

### 8.2 선택 달성 목표
- 패링 시스템 구현
- 부드러운 애니메이션 전환
- 시각/청각 피드백 추가

---

## 9. 제약사항 및 고려사항

### 9.1 제약사항
- 프로젝트 규모: 소규모 (개인 프로젝트)
- 개발 기간: 5-7일 (고급 기능 포함 시)
- 리소스: 무료 에셋 활용

### 9.2 고려사항
- 코드 재사용성을 위한 컴포넌트 분리
- Inspector에서 조정 가능한 파라미터 제공
- Gizmos를 이용한 디버깅 기능
- 명확한 네이밍과 주석

---

## 10. 참고 자료

### 10.1 Unity 공식 문서
- NavMesh: https://docs.unity3d.com/Manual/nav-NavigationSystem.html
- Animation Events: https://docs.unity3d.com/Manual/script-AnimationWindowEvent.html
- Collision Detection: https://docs.unity3d.com/Manual/CollidersOverview.html

### 10.2 디자인 패턴
- State Pattern: https://gameprogrammingpatterns.com/state.html

---

## 변경 이력
- 2025-11-18: 초기 PRD 작성
