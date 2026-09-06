# Dogsoul

<img width="1346" height="760" alt="image" src="https://github.com/user-attachments/assets/fbc785ce-14c8-47ee-85cd-4b7dcf6cae07" />


> **Unity**와 **Photon**, **Firebase**를 활용한 멀티플레이 TPS ARPG 프로젝트입니다. 소울류 게임의 메커니즘을 지니되, SD 캐릭터, 빠른 액션, 멀티플레이 협업 요소를 추가하여 캐주얼하게 재해석했습니다.
- **프로젝트 구분**: 팀 프로젝트 (3명)
- **장르**: 멀티플레이 TPS ARPG
- **담당 역할**: 클라이언트 개발(플레이어, 몬스터, 전투), 기획, 팀장
- **개발 환경**: Unity 2022.3.56f1, C#, VSCode, Photon, Firebase
- **개발 기간**: 약 4개월 (2025.01.29~2025.05.16)
- **주요 링크**:
  - 🎬 [YouTube 발표 영상](https://youtu.be/OtntBaEmVDU)
  - 📋 [Google Drive 발표 자료](https://drive.google.com/file/d/1VLTU9QHJnw7QmHsDgh-fGPYmPtici6cO/view?usp=drive_link)

---

## 게임 소개
- 기존 소울류 게임의 묵직한 분위기를 캐주얼 게임의 가벼운 분위기와 접목하여, 누구나 부담 없이 진입하되 소울류 특유의 정교한 공방의 재미를 느낄 수 있도록 기획했습니다.
- 다양한 무기가 존재하고, 각 무기군마다 고유한 스탯과 콤보, 애니메이션 세트를 제공하여 전략적인 전투가 가능합니다.

---

## 핵심 구현 컨텐츠

### 1. StateMachineBehaviour 기반의 애니메이션 제어, AnimatorOverrideController를 활용한 애니메이터 교체
- **콤보 및 캔슬 시스템 구현**: 애니메이션 이벤트와 StateMachineBehaviour를 모두 활용하여 플레이어 로코모션을 제작했습니다. StateMachineBehaviour 내 normalizedTime을 활용해 콤보 입력 가능 구간(ComboStart~ComboEnd)과 선/후딜레이 캔슬 지점을 세밀하게 제어할 수 있도록 설계하여 콤보 및 캔슬 기능을 구현했습니다.
- **실시간 애니메이터 오버라이드 시스템 구현**: 무기 데이터 필드값으로 애니메이터 오버라이드 키를 전달하도록 하여 간편하게 런타임 애니메이터를 교체할 수 있도록 설계했습니다.
- **타입 안정성을 갖춘 애니메이션 파이프라인 설계**: 파라미터 문자열 하드코딩으로 인한 오탈자 버그를 방지하고자 Enum과 Dictionary 기반의 wrapper 클래스(AnimationHandler)를 별도 구현하여 애니메이션 상태 제어의 안정성과 가독성을 높였습니다.

다음은 주요 코드 요약 (콤보 공격 상태머신 제어, 애니메이터 오버라이드 과정) 입니다.
> ExitAttack.cs
``` c#
public class ExitAttack : StateMachineBehaviour
{
    public string triggerName;
    public float ComboStart = 0.3f;
    public float ComboEnd = 0.7f;

    public override void OnStateUpdate(Animator animator, AnimatorStateInfo stateInfo, int layerIndex)
    {
        float progress = stateInfo.normalizedTime;
        if (progress >= freeProgressRate && progress < 1.0f)
        {      
            animator.SetBool(INTERACTING_LABEL, false);
            animator.applyRootMotion = false;
        }
        else
        {
            animator.SetBool(INTERACTING_LABEL, true);
            animator.SetBool(BLOCKING_LABEL, true);
            animator.applyRootMotion = true;
        }

        if (progress >= ComboStart && progress <= ComboEnd)
        {
            animator.SetBool(CANDOCOMBO_LABEL, true);
        }
        else
        {
            animator.SetBool(CANDOCOMBO_LABEL, false);
        }
    }
}
```

> WeaponHolderSlot.cs
```c#
public virtual void LoadWeaponModel(WeaponStats weaponStats)
{
    UnloadWeaponAndDestroy();

    weapon = Instantiate(weaponStats.weaponPrefab) as GameObject;
    if (weapon != null)
        if (animationHandler != null)
            animationHandler.UpdateOverride(weaponStats.weaponType);

    currentWeaponModel = weapon;
}
```
<br>

### 2. 거리 기반 AI 및 NavMesh 기반 보스 패턴 파이프라인 구축
- **NavMesh 경로 기반**의 정확한 타겟 거리 계산: NavMeshAgent.CalculatePath로 모퉁이 사이의 실제 이동 경로 거리를 산출하여, 장애물 뒤에 있는 타겟에 대한 비정상적인 추적/공격 판단을 방지했습니다.
- 거리별 전투 **상태 머신(FSM)** 구축: 플레이어와의 거리에 따라 근접 공격, 특수 공격(점프 공격, 투사체 발사), 추적(Chase), 순찰(Patrol), 복귀(BackToSpawn) 상태를 체계적으로 전환하도록 설계했습니다.
- **애니메이션 및 코루틴 기반**의 특수 패턴: 타깃이 멀리 떨어져 있을 때 보스의 점프 공격은 애니메이션 길이 만큼의 지속시간을 지니는 코루틴과 포물선 보간을 통해 역동감 있게 플레이어를 추적할 수 있게 설계했습니다.

![인게임](/gifs/jump.gif)

다음은 주요 코드 요약 (점프 공격) 입니다.
> BossController.cs
```c#
IEnumerator JumpCorutine()
{
    Vector3 startPos = transform.position;
    Vector3 targetPos = target.position;

    float jumpHeight = 5f;
    float duration = 1.1f;
    float time = 0f;

    while (time < duration)
    {
        float t = time / duration;
        // 포물선 보간
        Vector3 currentPos = Vector3.Lerp(startPos, targetPos, t);
        currentPos.y += Mathf.Sin(Mathf.PI * t) * jumpHeight;

        transform.position = currentPos;

        time += Time.deltaTime;
        yield return null;
    }

    // 착지 처리
    OnLand();

    // 점프 쿨타임
    yield return new WaitForSeconds(3f);
}
```


<br>

### 3. 무기의 능력치를 전투 능력치로 사용하는 인터랙티브 전투 파이프라인
- **무기 스탯 기반**의 슈퍼아머/피격 판정: 플레이어와 적의 공격이 동시에 이루어진 경우, 공격 주체 간 무기의 강인도 (tenacity) 수치를 비교하여, 피격자가 더 높은 강인도로 공격 중일 경우 피격 판정 및 경직을 무시하는 상쇄 메커니즘을 구현했습니다.
- **애니메이션 이벤트**를 통한 정확한 콜라이더 제어: 무기의 DamageCollider 활성화/비활성화 시점과 궤적 애니메이션(Trail)을 동기화하고, 피격 성공 시 본인의 공격 콜라이더를 즉시 닫아 연타 및 캔슬 오류를 방지했습니다.
- 상태 래퍼 기반의 유연한 피격 리액션 처리: 피격 시 enum 기반 피격 상태(Hit, Stun, Invincible, Die)를 전환하고, 공중 피격 캔슬, 피격 이펙트 생성, 회피를 통한 무적 상태 부여 등 **소울류 게임의 기초적인 전투 파이프라인을 거의 동일하게 구현**했습니다.

![인게임](/gifs/dodge.gif)

다음은 주요 코드 요약 (플레이어 피격 함수) 입니다.
> PlayerHealth.cs
```c#
public void TakeDamage(float damage, DamageCollider attackerWeapon, Vector3 contactPos, ParticleSystem hitEffect, bool isStun)
{
    DamageCollider myWeaponCollider = GetComponentInChildren<DamageCollider>();

    #region CancelCases
    // #1: when entity is invincible
    if (PlayerState.Instance.GetCurrentState() == PlayerState.State.Invincible || PlayerState.Instance.GetCurrentState() == PlayerState.State.Die) return;

    // #2: when my tenacity is larger than attacker's
    if (attackerWeapon != null && myWeaponCollider != null)
    {
        if (animationHandler.GetBool(AnimationHandler.AnimParam.Attacking) &&
        myWeaponCollider.tenacity > attackerWeapon.tenacity) return;
    }
    #endregion
}
```

<br>

### 4. 기타 핵심 구현 컨텐츠
- OverlapSphere를 활용한 락온 시스템: OverlapSpehre로 주위 적을 탐지하여 가장 가까운 순서대로 리스트에 저장하고, 플레이어의 forward 벡터로 시야 각도를 계산하여 락온 범위안에 들어온 적을 바라보는 락온 기능을 구현했습니다.
- MeshRenderer와 Coroutine을 활용한 잔상효과: MeshRenderer로 플레이어의 메쉬를 캐싱하여 회색 머티리얼을 부여하고 코루틴으로 일정 주기마다 메쉬 잔상 오브젝트를 생성하게 하여 회피에 사용되는 시각 효과를 제작했습니다.
- InputSystem을 활용한 입력 구현: InputManager나 구형 유니티 인풋 시스템이 아닌, InputSystem 패키지에 맞춰 플레이어 입력을 받아 컨트롤러에게 value.isPreseed 등의 값을 전달하도록 구현했습니다.

--- 

## 트러블슈팅

### 1. Git 히스토리 유실 사고 발생
- 문제 상황: Git 명령어 숙련도가 부족했던 시기에 작업 브랜치의 커밋 히스토리를 강제로 덮어쓰면서, 작업 중이던 주요 기능 코드가 유실되는 사고가 발생했습니다.
- 대처 및 해결: 팀원에게 상황을 즉시 공유하고 팀원의 최신 커밋을 다시 불러온 뒤, 한 노트북 앞에 함께 앉아 제가 수정한 내역들을 직접 설명해 가며 실시간으로 코드를 재구현하고 복구했습니다.
- 배운 점 및 개선 조치: git push --force 사용을 엄격히 금지하고, 작업중인 브랜치의 히스토리를 주기적으로 확인하며 개발에 임했습니다.

### 2. 플레이어 움직임 방식(Rigidbody)의 문제점 개선
- 문제 상황: 초기에 리지드바디를 이용해서 플레이어 구현 시 빠른 이동이나 모퉁이 회전 시, 캐릭터가 벽을 뚫고 지나가거나 지형 밑으로 낙하하는 충돌 뚫림(Tunneling) 현상이 지속적으로 발생했습니다.
- 원인 분석: Rigidbody 기반 이동은 물리 연산 주기(FixedUpdate)마다 힘이나 속도를 적용하는 방식이기 때문에, 프레임 드랍이나 높은 속도에서 물리 콜라이더 연산이 충돌체를 지나쳐 버리는 터널링 문제가 발생하는 것이 원인이었습니다.
- 대처 및 해결: 물리 연산 오차에 민감한 Rigidbody 대신, 캐릭터 전용 레이캐스트 및 경사면 처리 기능이 내장된 CharacterController 스크립트 구조로 전환하고, .Move() 함수를 활용하여 프레임 단위의 정확한 이동량을 제어하여, 경사면 이동 및 충돌 판정의 안정성을 대폭 개선하여 벽 뚫림 현상을 해결했습니다.
- 배운 점 및 개선 조치: Rigidbody와 CharacterController의 작동 방식을 자세히 알게 되었고, 게임엔진에서 객체의 움직임을 나타내기 위한 방법을 고민해보는 시간을 가질 수 있었습니다.

---

## 개발 회고 및 성찰

- **모듈화 중심의 확장성 있는 코드 설계**:  협업 환경에서 누구나 접근하기 편하고 수정과 확장이 자유로운 코드를 만드는 것이 개발자의 핵심 덕목이라 생각해 모듈화를 이번 첫 팀 프로젝트의 주요 컨셉으로 잡았습니다. 각 모듈의 책임을 PlayerController, PlayerHealth, PlayerState, InputHandler 등 컴포넌트 단위로 독립시켜 팀원이 내 코드로 쉽게 컨텐츠를 확장할 수 있게 만들면서, 왜 가독성과 모듈화 구조가 유지보수에 중요한지 깊게 체감할 수 있었습니다.

- **동료와의 실시간 커뮤니케이션과 협업**: Git 사고로 코드가 유실되었을 때, 솔직하게 상황을 공유하고 팀원의 최신 커밋을 불러와 한 노트북 앞에 같이 앉아 실시간으로 소통하며 복구해 나갔습니다. 이 과정을 통해 예기치 못한 문제가 생겼을 때의 대처 능력뿐만 아니라, 단순히 혼자 코드를 잘 짜는 것을 넘어 팀원과 맞춰가며 작업하는 유연한 협업 방식의 중요성을 깊게 느꼈습니다. 이러한 맥락에서 왜 유지보수성과 가독성이 중요한지 알 수 있었고, 단순히 높은 기술력 뿐만 아니라 팀원의 스타일을 고려하여 코드를 작성할 줄 아는 유연함 역시 개발자에게 필요한 역량이라는 것을 깨달았습니다.
