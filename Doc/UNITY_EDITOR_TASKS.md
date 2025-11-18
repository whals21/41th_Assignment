# Unity Editor 작업 가이드
## 3D 액션 게임 - 적 AI 및 전투 시스템 구현

이 문서는 PRD.md에 정의된 3D 액션 게임을 Unity 에디터에서 단계별로 구현하는 가이드입니다.

---

## 목차
1. [프로젝트 초기 설정](#1-프로젝트-초기-설정)
2. [Phase 1: 기본 플레이어 시스템](#phase-1-기본-플레이어-시스템)
3. [Phase 2: 플레이어 전투 시스템](#phase-2-플레이어-전투-시스템)
4. [Phase 3: 적 AI 기본 구조](#phase-3-적-ai-기본-구조)
5. [Phase 4: 적 전투 시스템](#phase-4-적-전투-시스템)
6. [Phase 5: 고급 기능 (패링)](#phase-5-고급-기능-패링)
7. [테스트 및 디버깅](#테스트-및-디버깅)

---

## 1. 프로젝트 초기 설정

### 1.1 Unity 프로젝트 설정 확인
- [ ] Unity 버전: 2021.3 LTS 이상 확인
- [ ] 프로젝트 템플릿: 3D (URP 권장)
- [ ] Input System: 신규 Input System 설치 (Package Manager)

### 1.2 프로젝트 구조 생성
Unity Project 창에서 다음 폴더 구조를 생성합니다:

```
Assets/
├── Scripts/
│   ├── Player/
│   ├── Enemy/
│   ├── Combat/
│   └── Common/
├── Prefabs/
│   ├── Player/
│   ├── Enemy/
│   └── Effects/
├── Materials/
├── Animations/
│   ├── Player/
│   └── Enemy/
├── Scenes/
└── Resources/
```

**작업 방법:**
1. Project 창에서 우클릭 → Create → Folder
2. 위 구조대로 폴더 생성

### 1.3 씬 설정
- [ ] 새 씬 생성: `Scenes/GameScene`
- [ ] Directional Light 설정 확인
- [ ] Main Camera 설정 (Position: 0, 5, -10, Rotation: 30, 0, 0)

### 1.4 기본 레벨 구성
**바닥 생성:**
1. Hierarchy → 우클릭 → 3D Object → Plane
2. 이름: "Ground"
3. Transform: Position (0, 0, 0), Scale (5, 1, 5)
4. Material 생성: `Materials/GroundMaterial`
5. Ground에 Material 적용

**벽/장애물 추가 (선택):**
1. Hierarchy → 우클릭 → 3D Object → Cube
2. 간단한 레벨 지오메트리 구성

---

## Phase 1: 기본 플레이어 시스템

### 1.1 플레이어 캐릭터 생성

#### 1.1.1 플레이어 오브젝트 생성
1. Hierarchy → 우클릭 → 3D Object → Capsule
2. 이름: "Player"
3. Transform:
   - Position: (0, 1, 0)
   - Scale: (1, 1, 1)

4. Add Component → Rigidbody
   - Mass: 1
   - Drag: 0
   - Angular Drag: 0.05
   - Use Gravity: ✓
   - Is Kinematic: ☐
   - Constraints: Freeze Rotation X, Y, Z ✓

**또는 CharacterController 사용:**
1. Rigidbody 대신 Add Component → Character Controller
   - Center: (0, 1, 0)
   - Radius: 0.5
   - Height: 2

#### 1.1.2 플레이어 시각 구성
자식 오브젝트로 간단한 모델 추가:
1. Player 우클릭 → 3D Object → Cylinder (몸통)
   - Name: "Body"
   - Scale: (0.5, 0.8, 0.5)
   - Position: (0, 0, 0)

2. Player 우클릭 → 3D Object → Sphere (머리)
   - Name: "Head"
   - Scale: (0.4, 0.4, 0.4)
   - Position: (0, 0.9, 0)

3. Player 우클릭 → 3D Object → Cube (검 - 나중에 사용)
   - Name: "Weapon"
   - Scale: (0.1, 0.8, 0.1)
   - Position: (0.5, 0.5, 0)
   - 일단 비활성화

#### 1.1.3 Tag 및 Layer 설정
1. Player 선택 → Inspector 상단
2. Tag: "Player" (없으면 Add Tag로 생성)
3. Layer: "Player" (없으면 생성)

### 1.2 플레이어 이동 스크립트 작성

#### 1.2.1 스크립트 생성
1. `Scripts/Player/` 폴더에서 우클릭
2. Create → C# Script
3. 이름: "PlayerController"

#### 1.2.2 코드 작성
```csharp
using UnityEngine;

public class PlayerController : MonoBehaviour
{
    [Header("Movement Settings")]
    [SerializeField] private float moveSpeed = 5f;
    [SerializeField] private float jumpForce = 5f;
    [SerializeField] private float gravity = -9.81f;

    [Header("Ground Check")]
    [SerializeField] private Transform groundCheck;
    [SerializeField] private float groundDistance = 0.4f;
    [SerializeField] private LayerMask groundMask;

    private CharacterController controller;
    private Vector3 velocity;
    private bool isGrounded;

    private void Start()
    {
        controller = GetComponent<CharacterController>();

        // Ground Check 오브젝트가 없으면 생성
        if (groundCheck == null)
        {
            GameObject gc = new GameObject("GroundCheck");
            gc.transform.SetParent(transform);
            gc.transform.localPosition = new Vector3(0, -1f, 0);
            groundCheck = gc.transform;
        }
    }

    private void Update()
    {
        CheckGround();
        HandleMovement();
        HandleJump();
        ApplyGravity();
    }

    private void CheckGround()
    {
        isGrounded = Physics.CheckSphere(groundCheck.position, groundDistance, groundMask);

        if (isGrounded && velocity.y < 0)
        {
            velocity.y = -2f; // 작은 음수값으로 지면에 붙어있게
        }
    }

    private void HandleMovement()
    {
        // 입력 받기
        float horizontal = Input.GetAxis("Horizontal"); // A, D 또는 ←, →
        float vertical = Input.GetAxis("Vertical");     // W, S 또는 ↑, ↓

        // 카메라 기준 이동 방향 계산
        Vector3 direction = new Vector3(horizontal, 0f, vertical).normalized;

        if (direction.magnitude >= 0.1f)
        {
            // 이동
            Vector3 move = direction * moveSpeed * Time.deltaTime;
            controller.Move(move);

            // 캐릭터 회전 (이동 방향으로)
            transform.forward = direction;
        }
    }

    private void HandleJump()
    {
        if (Input.GetButtonDown("Jump") && isGrounded)
        {
            velocity.y = Mathf.Sqrt(jumpForce * -2f * gravity);
        }
    }

    private void ApplyGravity()
    {
        velocity.y += gravity * Time.deltaTime;
        controller.Move(velocity * Time.deltaTime);
    }

    // Gizmos로 Ground Check 영역 시각화
    private void OnDrawGizmosSelected()
    {
        if (groundCheck != null)
        {
            Gizmos.color = Color.yellow;
            Gizmos.DrawWireSphere(groundCheck.position, groundDistance);
        }
    }
}
```

#### 1.2.3 스크립트 적용
1. PlayerController 스크립트를 Player 오브젝트에 드래그
2. Inspector에서 설정:
   - Move Speed: 5
   - Jump Force: 5
   - Gravity: -9.81
   - Ground Mask: "Default" 또는 "Ground" 레이어 선택

### 1.3 카메라 설정

#### 1.3.1 기본 3인칭 카메라
1. Main Camera를 Player의 자식으로 설정
2. Position: (0, 3, -5)
3. Rotation: (20, 0, 0)

#### 1.3.2 카메라 스크립트 (선택)
더 나은 카메라 컨트롤을 위해:

`Scripts/Player/CameraFollow.cs`:
```csharp
using UnityEngine;

public class CameraFollow : MonoBehaviour
{
    [SerializeField] private Transform target;
    [SerializeField] private Vector3 offset = new Vector3(0, 3, -5);
    [SerializeField] private float smoothSpeed = 0.125f;

    private void LateUpdate()
    {
        if (target == null) return;

        Vector3 desiredPosition = target.position + offset;
        Vector3 smoothedPosition = Vector3.Lerp(transform.position, desiredPosition, smoothSpeed);
        transform.position = smoothedPosition;

        transform.LookAt(target);
    }
}
```

### 1.4 Phase 1 테스트
- [ ] Play 버튼을 눌러 게임 실행
- [ ] WASD 키로 플레이어 이동 확인
- [ ] Space 바로 점프 확인
- [ ] 바닥에서 떨어지지 않는지 확인

---

## Phase 2: 플레이어 전투 시스템

### 2.1 체력 시스템 구현

#### 2.1.1 HealthSystem 스크립트 생성
`Scripts/Common/HealthSystem.cs`:
```csharp
using UnityEngine;
using UnityEngine.Events;

public class HealthSystem : MonoBehaviour
{
    [Header("Health Settings")]
    [SerializeField] private int maxHealth = 100;
    private int currentHealth;

    [Header("Events")]
    public UnityEvent<int> OnHealthChanged;
    public UnityEvent OnDeath;

    private void Start()
    {
        currentHealth = maxHealth;
    }

    public void TakeDamage(int damage)
    {
        currentHealth -= damage;
        currentHealth = Mathf.Max(currentHealth, 0);

        OnHealthChanged?.Invoke(currentHealth);

        if (currentHealth <= 0)
        {
            Die();
        }
    }

    public void Heal(int amount)
    {
        currentHealth += amount;
        currentHealth = Mathf.Min(currentHealth, maxHealth);
        OnHealthChanged?.Invoke(currentHealth);
    }

    private void Die()
    {
        OnDeath?.Invoke();
        Debug.Log($"{gameObject.name} has died!");
    }

    public int GetCurrentHealth() => currentHealth;
    public int GetMaxHealth() => maxHealth;
    public bool IsAlive() => currentHealth > 0;
}
```

#### 2.1.2 Player에 HealthSystem 추가
1. Player 오브젝트 선택
2. Add Component → Health System
3. Max Health: 100

### 2.2 공격 시스템 구현

#### 2.2.1 PlayerCombat 스크립트 생성
`Scripts/Player/PlayerCombat.cs`:
```csharp
using UnityEngine;

public class PlayerCombat : MonoBehaviour
{
    [Header("Attack Settings")]
    [SerializeField] private int attackDamage = 20;
    [SerializeField] private float attackCooldown = 1f;
    [SerializeField] private GameObject weaponCollider;

    private float lastAttackTime;
    private bool isAttacking;
    private Animator animator; // 나중에 추가

    private void Start()
    {
        animator = GetComponent<Animator>();

        // 무기 콜라이더 비활성화
        if (weaponCollider != null)
        {
            weaponCollider.SetActive(false);
        }
    }

    private void Update()
    {
        HandleAttackInput();
    }

    private void HandleAttackInput()
    {
        // 마우스 좌클릭 또는 F 키
        if (Input.GetMouseButtonDown(0) || Input.GetKeyDown(KeyCode.F))
        {
            if (Time.time >= lastAttackTime + attackCooldown && !isAttacking)
            {
                Attack();
            }
        }
    }

    private void Attack()
    {
        isAttacking = true;
        lastAttackTime = Time.time;

        Debug.Log("Player attacks!");

        // 애니메이션 재생 (나중에 추가)
        // animator?.SetTrigger("Attack");

        // 무기 콜라이더 활성화 (애니메이션 이벤트로 호출 예정)
        EnableWeaponCollider();

        // 0.5초 후 공격 종료
        Invoke(nameof(EndAttack), 0.5f);
    }

    public void EnableWeaponCollider()
    {
        if (weaponCollider != null)
        {
            weaponCollider.SetActive(true);
            Invoke(nameof(DisableWeaponCollider), 0.3f);
        }
    }

    public void DisableWeaponCollider()
    {
        if (weaponCollider != null)
        {
            weaponCollider.SetActive(false);
        }
    }

    private void EndAttack()
    {
        isAttacking = false;
    }

    public int GetAttackDamage() => attackDamage;
}
```

### 2.3 무기 피격 판정 설정

#### 2.3.1 무기 오브젝트 설정
1. Player → Weapon 오브젝트 활성화
2. Add Component → Box Collider
   - Is Trigger: ✓
   - Size: (0.2, 1, 0.2)

3. Player → Weapon에 Tag 설정: "PlayerWeapon"

#### 2.3.2 DamageDealer 스크립트 생성
`Scripts/Combat/DamageDealer.cs`:
```csharp
using UnityEngine;

public class DamageDealer : MonoBehaviour
{
    [SerializeField] private int damage = 20;
    [SerializeField] private string targetTag = "Enemy"; // 공격할 대상 태그

    private void OnTriggerEnter(Collider other)
    {
        if (other.CompareTag(targetTag))
        {
            // 상대방에게 데미지 전달
            HealthSystem healthSystem = other.GetComponent<HealthSystem>();
            if (healthSystem != null)
            {
                healthSystem.TakeDamage(damage);
                Debug.Log($"Hit {other.name} for {damage} damage!");
            }
        }
    }

    public void SetDamage(int newDamage)
    {
        damage = newDamage;
    }
}
```

#### 2.3.3 무기에 DamageDealer 추가
1. Player → Weapon 선택
2. Add Component → Damage Dealer
3. Damage: 20
4. Target Tag: "Enemy"

#### 2.3.4 PlayerCombat과 연결
1. Player 선택
2. PlayerCombat 컴포넌트에서
3. Weapon Collider에 "Weapon" 오브젝트 드래그

### 2.4 Layer 설정
충돌 매트릭스 설정:
1. Edit → Project Settings → Physics
2. Layer Collision Matrix에서:
   - Player와 PlayerWeapon 충돌 해제
   - PlayerWeapon과 Enemy 충돌 활성화

### 2.5 Phase 2 테스트
- [ ] 플레이어 공격 입력 (마우스 좌클릭 또는 F)
- [ ] 무기가 활성화/비활성화 되는지 확인
- [ ] 쿨타임이 작동하는지 확인
- [ ] Console에 "Player attacks!" 메시지 확인

---

## Phase 3: 적 AI 기본 구조

### 3.1 NavMesh 설정

#### 3.1.1 NavMesh 베이크
1. Window → AI → Navigation
2. Baking 탭 선택
3. 설정:
   - Agent Radius: 0.5
   - Agent Height: 2
   - Max Slope: 45
   - Step Height: 0.4
4. Ground 오브젝트 선택 → Navigation Static 체크
5. Bake 버튼 클릭

### 3.2 적 캐릭터 생성

#### 3.2.1 적 오브젝트 생성
1. Hierarchy → 우클릭 → 3D Object → Capsule
2. 이름: "Enemy"
3. Transform:
   - Position: (5, 1, 0)
   - Scale: (1, 1, 1)

4. Add Component → Nav Mesh Agent
   - Speed: 3.5
   - Angular Speed: 120
   - Acceleration: 8
   - Stopping Distance: 2
   - Auto Braking: ✓

5. Add Component → Health System
   - Max Health: 100

6. Tag: "Enemy" (없으면 생성)
7. Layer: "Enemy" (없으면 생성)

#### 3.2.2 적 시각 구성
1. Enemy → 자식 → Capsule (Body)
   - Material: 빨간색 Material 생성 및 적용

### 3.3 상태 패턴 구조 구현

#### 3.3.1 IEnemyState 인터페이스
`Scripts/Enemy/IEnemyState.cs`:
```csharp
public interface IEnemyState
{
    void EnterState(EnemyAI enemy);
    void UpdateState(EnemyAI enemy);
    void ExitState(EnemyAI enemy);
}
```

#### 3.3.2 PatrolState
`Scripts/Enemy/PatrolState.cs`:
```csharp
using UnityEngine;

public class PatrolState : IEnemyState
{
    private int currentWaypointIndex;

    public void EnterState(EnemyAI enemy)
    {
        Debug.Log($"{enemy.name} entered Patrol state");
        enemy.Agent.speed = enemy.PatrolSpeed;

        if (enemy.Waypoints.Length > 0)
        {
            MoveToNextWaypoint(enemy);
        }
    }

    public void UpdateState(EnemyAI enemy)
    {
        // 플레이어 감지 체크
        float distanceToPlayer = Vector3.Distance(enemy.transform.position, enemy.Player.position);

        if (distanceToPlayer <= enemy.ChaseRange)
        {
            enemy.ChangeState(enemy.ChaseState);
            return;
        }

        // 웨이포인트 도착 체크
        if (enemy.Waypoints.Length > 0)
        {
            if (!enemy.Agent.pathPending && enemy.Agent.remainingDistance < 0.5f)
            {
                MoveToNextWaypoint(enemy);
            }
        }
        else
        {
            // 웨이포인트가 없으면 랜덤 배회
            if (!enemy.Agent.pathPending && enemy.Agent.remainingDistance < 0.5f)
            {
                Vector3 randomPoint = enemy.transform.position + Random.insideUnitSphere * 10f;
                randomPoint.y = enemy.transform.position.y;
                enemy.Agent.SetDestination(randomPoint);
            }
        }
    }

    public void ExitState(EnemyAI enemy)
    {
        Debug.Log($"{enemy.name} exited Patrol state");
    }

    private void MoveToNextWaypoint(EnemyAI enemy)
    {
        if (enemy.Waypoints.Length == 0) return;

        enemy.Agent.SetDestination(enemy.Waypoints[currentWaypointIndex].position);
        currentWaypointIndex = (currentWaypointIndex + 1) % enemy.Waypoints.Length;
    }
}
```

#### 3.3.3 ChaseState
`Scripts/Enemy/ChaseState.cs`:
```csharp
using UnityEngine;

public class ChaseState : IEnemyState
{
    public void EnterState(EnemyAI enemy)
    {
        Debug.Log($"{enemy.name} entered Chase state");
        enemy.Agent.speed = enemy.ChaseSpeed;
    }

    public void UpdateState(EnemyAI enemy)
    {
        float distanceToPlayer = Vector3.Distance(enemy.transform.position, enemy.Player.position);

        // 공격 범위 진입
        if (distanceToPlayer <= enemy.AttackRange)
        {
            enemy.ChangeState(enemy.AttackState);
            return;
        }

        // 추적 범위 이탈
        if (distanceToPlayer > enemy.ChaseRange)
        {
            enemy.ChangeState(enemy.PatrolState);
            return;
        }

        // 플레이어 추적
        enemy.Agent.SetDestination(enemy.Player.position);
    }

    public void ExitState(EnemyAI enemy)
    {
        Debug.Log($"{enemy.name} exited Chase state");
    }
}
```

#### 3.3.4 AttackState
`Scripts/Enemy/AttackState.cs`:
```csharp
using UnityEngine;

public class AttackState : IEnemyState
{
    private float lastAttackTime;

    public void EnterState(EnemyAI enemy)
    {
        Debug.Log($"{enemy.name} entered Attack state");
        enemy.Agent.speed = 0; // 공격 중 정지
    }

    public void UpdateState(EnemyAI enemy)
    {
        float distanceToPlayer = Vector3.Distance(enemy.transform.position, enemy.Player.position);

        // 공격 범위 이탈
        if (distanceToPlayer > enemy.AttackRange)
        {
            enemy.ChangeState(enemy.ChaseState);
            return;
        }

        // 플레이어 바라보기
        Vector3 direction = (enemy.Player.position - enemy.transform.position).normalized;
        direction.y = 0;
        enemy.transform.forward = direction;

        // 공격 쿨타임 체크
        if (Time.time >= lastAttackTime + enemy.AttackCooldown)
        {
            PerformAttack(enemy);
            lastAttackTime = Time.time;
        }
    }

    public void ExitState(EnemyAI enemy)
    {
        Debug.Log($"{enemy.name} exited Attack state");
    }

    private void PerformAttack(EnemyAI enemy)
    {
        Debug.Log($"{enemy.name} attacks the player!");

        // 애니메이션 재생 (나중에 추가)
        // enemy.Animator?.SetTrigger("Attack");

        // 플레이어에게 데미지
        if (enemy.Player != null)
        {
            HealthSystem playerHealth = enemy.Player.GetComponent<HealthSystem>();
            if (playerHealth != null)
            {
                playerHealth.TakeDamage(enemy.AttackDamage);
            }
        }
    }
}
```

#### 3.3.5 EnemyAI 메인 컨트롤러
`Scripts/Enemy/EnemyAI.cs`:
```csharp
using UnityEngine;
using UnityEngine.AI;

public class EnemyAI : MonoBehaviour
{
    [Header("References")]
    [SerializeField] private Transform player;
    [SerializeField] private Transform[] waypoints;

    [Header("Movement Settings")]
    [SerializeField] private float patrolSpeed = 2f;
    [SerializeField] private float chaseSpeed = 5f;

    [Header("Detection Settings")]
    [SerializeField] private float chaseRange = 10f;
    [SerializeField] private float attackRange = 2f;

    [Header("Combat Settings")]
    [SerializeField] private int attackDamage = 10;
    [SerializeField] private float attackCooldown = 2f;

    // 상태들
    public IEnemyState PatrolState { get; private set; }
    public IEnemyState ChaseState { get; private set; }
    public IEnemyState AttackState { get; private set; }

    private IEnemyState currentState;

    // 컴포넌트
    private NavMeshAgent agent;
    public NavMeshAgent Agent => agent;

    // Properties
    public Transform Player => player;
    public Transform[] Waypoints => waypoints;
    public float PatrolSpeed => patrolSpeed;
    public float ChaseSpeed => chaseSpeed;
    public float ChaseRange => chaseRange;
    public float AttackRange => attackRange;
    public int AttackDamage => attackDamage;
    public float AttackCooldown => attackCooldown;

    private void Start()
    {
        agent = GetComponent<NavMeshAgent>();

        // 플레이어 자동 찾기
        if (player == null)
        {
            GameObject playerObj = GameObject.FindGameObjectWithTag("Player");
            if (playerObj != null)
            {
                player = playerObj.transform;
            }
        }

        // 상태 초기화
        PatrolState = new PatrolState();
        ChaseState = new ChaseState();
        AttackState = new AttackState();

        // 초기 상태 설정
        currentState = PatrolState;
        currentState.EnterState(this);
    }

    private void Update()
    {
        currentState?.UpdateState(this);
    }

    public void ChangeState(IEnemyState newState)
    {
        currentState?.ExitState(this);
        currentState = newState;
        currentState?.EnterState(this);
    }

    // Gizmos로 범위 시각화
    private void OnDrawGizmosSelected()
    {
        // 추적 범위
        Gizmos.color = Color.yellow;
        Gizmos.DrawWireSphere(transform.position, chaseRange);

        // 공격 범위
        Gizmos.color = Color.red;
        Gizmos.DrawWireSphere(transform.position, attackRange);

        // 웨이포인트 연결선
        if (waypoints != null && waypoints.Length > 1)
        {
            Gizmos.color = Color.blue;
            for (int i = 0; i < waypoints.Length; i++)
            {
                if (waypoints[i] != null)
                {
                    Vector3 current = waypoints[i].position;
                    Vector3 next = waypoints[(i + 1) % waypoints.Length].position;
                    Gizmos.DrawLine(current, next);
                    Gizmos.DrawWireSphere(current, 0.3f);
                }
            }
        }
    }
}
```

### 3.4 웨이포인트 설정

#### 3.4.1 웨이포인트 생성
1. Hierarchy → 빈 오브젝트 생성 → 이름: "Waypoints"
2. Waypoints 자식으로 빈 오브젝트 생성 (4개)
   - Waypoint1: (5, 0, 5)
   - Waypoint2: (5, 0, -5)
   - Waypoint3: (-5, 0, -5)
   - Waypoint4: (-5, 0, 5)

#### 3.4.2 Enemy에 웨이포인트 연결
1. Enemy 선택
2. EnemyAI 컴포넌트 → Waypoints
3. Size: 4
4. 각 Element에 Waypoint1~4 드래그

### 3.5 Phase 3 테스트
- [ ] Play 버튼 클릭
- [ ] 적이 웨이포인트를 순찰하는지 확인
- [ ] 플레이어가 추적 범위에 들어가면 추적하는지 확인
- [ ] Scene 뷰에서 Gizmos로 범위 확인

---

## Phase 4: 적 전투 시스템

### 4.1 적 무기 설정

#### 4.1.1 적 무기 생성
1. Enemy → 자식 → 3D Object → Cube
2. 이름: "EnemyWeapon"
3. Transform:
   - Position: (0.5, 0.5, 0)
   - Scale: (0.1, 0.8, 0.1)

4. Add Component → Box Collider
   - Is Trigger: ✓
   - Size: (2, 1.2, 2) (넓게 설정)

5. Add Component → Damage Dealer
   - Damage: 10
   - Target Tag: "Player"

6. Tag: "EnemyWeapon"
7. 초기 비활성화: GameObject 체크 해제

### 4.2 EnemyCombat 스크립트 생성
`Scripts/Enemy/EnemyCombat.cs`:
```csharp
using UnityEngine;

public class EnemyCombat : MonoBehaviour
{
    [SerializeField] private GameObject weaponCollider;

    public void EnableWeaponCollider()
    {
        if (weaponCollider != null)
        {
            weaponCollider.SetActive(true);
        }
    }

    public void DisableWeaponCollider()
    {
        if (weaponCollider != null)
        {
            weaponCollider.SetActive(false);
        }
    }
}
```

### 4.3 AttackState 업데이트
AttackState.cs의 PerformAttack 수정:
```csharp
private void PerformAttack(EnemyAI enemy)
{
    Debug.Log($"{enemy.name} attacks the player!");

    // 적 무기 활성화
    EnemyCombat combat = enemy.GetComponent<EnemyCombat>();
    if (combat != null)
    {
        combat.EnableWeaponCollider();
        // 0.3초 후 비활성화
        enemy.StartCoroutine(DisableWeaponAfterDelay(combat, 0.3f));
    }
}

private System.Collections.IEnumerator DisableWeaponAfterDelay(EnemyCombat combat, float delay)
{
    yield return new WaitForSeconds(delay);
    combat.DisableWeaponCollider();
}
```

EnemyAI.cs에도 Coroutine을 위한 public 접근 제공.

### 4.4 Phase 4 테스트
- [ ] 적이 공격 범위에서 공격하는지 확인
- [ ] 플레이어가 피격되고 체력이 감소하는지 확인
- [ ] Console에서 데미지 로그 확인

---

## Phase 5: 고급 기능 (패링)

### 5.1 PlayerCombat에 패링 추가

#### 5.1.1 PlayerCombat 확장
```csharp
[Header("Parry Settings")]
[SerializeField] private float parryWindow = 0.3f;
[SerializeField] private float parryCooldown = 1f;

private bool isParrying;
private float parryStartTime;
private float lastParryTime;

private void Update()
{
    HandleAttackInput();
    HandleParryInput();
    UpdateParry();
}

private void HandleParryInput()
{
    // 마우스 우클릭 또는 Shift
    if (Input.GetMouseButtonDown(1) || Input.GetKeyDown(KeyCode.LeftShift))
    {
        if (Time.time >= lastParryTime + parryCooldown && !isParrying)
        {
            StartParry();
        }
    }
}

private void StartParry()
{
    isParrying = true;
    parryStartTime = Time.time;
    lastParryTime = Time.time;

    Debug.Log("Parry started!");
    // 패링 애니메이션 재생
}

private void UpdateParry()
{
    if (isParrying)
    {
        if (Time.time >= parryStartTime + parryWindow)
        {
            isParrying = false;
            Debug.Log("Parry window ended");
        }
    }
}

public bool IsParrying()
{
    return isParrying;
}
```

### 5.2 DamageDealer 수정 (패링 체크)
```csharp
private void OnTriggerEnter(Collider other)
{
    if (other.CompareTag(targetTag))
    {
        // 플레이어가 패링 중인지 체크
        if (targetTag == "Player")
        {
            PlayerCombat combat = other.GetComponent<PlayerCombat>();
            if (combat != null && combat.IsParrying())
            {
                Debug.Log("Attack parried!");
                ParrySuccess(other);
                return;
            }
        }

        // 일반 데미지 처리
        HealthSystem healthSystem = other.GetComponent<HealthSystem>();
        if (healthSystem != null)
        {
            healthSystem.TakeDamage(damage);
            Debug.Log($"Hit {other.name} for {damage} damage!");
        }
    }
}

private void ParrySuccess(Collider player)
{
    // 패링 성공 효과
    Debug.Log("Parry successful!");

    // 적을 경직시키는 로직 추가 가능
    // 예: 공격자(적)에게 스턴 효과
}
```

### 5.3 Phase 5 테스트
- [ ] 패리 입력 (우클릭 또는 Shift)
- [ ] 패링 타이밍 윈도우 테스트
- [ ] 적 공격을 패링했을 때 데미지가 안 들어오는지 확인

---

## 테스트 및 디버깅

### 디버그 도구

#### 1. DebugManager 스크립트
`Scripts/Common/DebugManager.cs`:
```csharp
using UnityEngine;

public class DebugManager : MonoBehaviour
{
    [SerializeField] private bool showDebugInfo = true;

    private void OnGUI()
    {
        if (!showDebugInfo) return;

        GUILayout.BeginArea(new Rect(10, 10, 300, 500));
        GUILayout.Box("Debug Info");

        // 플레이어 정보
        GameObject player = GameObject.FindGameObjectWithTag("Player");
        if (player != null)
        {
            HealthSystem health = player.GetComponent<HealthSystem>();
            if (health != null)
            {
                GUILayout.Label($"Player Health: {health.GetCurrentHealth()} / {health.GetMaxHealth()}");
            }
        }

        // 적 정보
        GameObject[] enemies = GameObject.FindGameObjectsWithTag("Enemy");
        GUILayout.Label($"Enemies: {enemies.Length}");

        foreach (var enemy in enemies)
        {
            EnemyAI ai = enemy.GetComponent<EnemyAI>();
            HealthSystem health = enemy.GetComponent<HealthSystem>();

            if (ai != null && health != null)
            {
                float dist = Vector3.Distance(player.transform.position, enemy.transform.position);
                GUILayout.Label($"{enemy.name}: HP {health.GetCurrentHealth()}, Dist {dist:F1}");
            }
        }

        GUILayout.EndArea();
    }
}
```

#### 2. 씬에 DebugManager 추가
1. Hierarchy → 빈 오브젝트 생성 → "GameManager"
2. Add Component → Debug Manager

### 최종 체크리스트

#### 플레이어
- [ ] 이동이 부드럽게 작동
- [ ] 점프가 정상 작동
- [ ] 공격 애니메이션 및 판정
- [ ] 체력 시스템
- [ ] 패링 (선택)

#### 적 AI
- [ ] 배회 상태 정상 작동
- [ ] 플레이어 추적
- [ ] 공격 실행
- [ ] 상태 전환이 자연스러움
- [ ] 체력 시스템

#### 전투
- [ ] 플레이어 공격이 적에게 데미지
- [ ] 적 공격이 플레이어에게 데미지
- [ ] 피격 판정이 정확함
- [ ] 쿨타임 작동

#### 비주얼
- [ ] Gizmos로 범위 확인 가능
- [ ] 디버그 UI 표시

---

## 추가 개선 사항 (선택)

### 애니메이션 추가
1. Animator Controller 생성
2. Idle, Walk, Attack, Death 애니메이션 추가
3. 스크립트에서 애니메이션 트리거

### UI 추가
1. Canvas 생성
2. 체력바 UI
3. 데미지 텍스트 표시

### 이펙트 추가
1. Particle System으로 공격 이펙트
2. 피격 이펙트
3. 사운드 추가

### 최적화
1. Object Pooling (적 생성)
2. LOD 설정
3. Occlusion Culling

---

## 문제 해결 (Troubleshooting)

### NavMesh 관련
**문제**: 적이 움직이지 않음
- NavMesh가 베이크되었는지 확인
- Agent가 NavMesh 위에 있는지 확인
- Obstacles 체크

### 충돌 감지 관련
**문제**: 공격이 적중하지 않음
- Collider가 Trigger로 설정되었는지 확인
- Layer Collision Matrix 확인
- Tag가 정확한지 확인

### 상태 전환 관련
**문제**: 상태가 전환되지 않음
- 거리 계산 확인 (Gizmos 활용)
- Debug.Log로 상태 확인
- Player 참조가 null이 아닌지 확인

---

## 다음 단계

이 가이드를 완료한 후:
1. 더 많은 적 추가
2. 다양한 적 타입 (원거리, 탱커 등)
3. 레벨 디자인 확장
4. 보스 전투
5. 저장/로드 시스템
6. 빌드 및 배포

---

**작성일**: 2025-11-18
**버전**: 1.0
