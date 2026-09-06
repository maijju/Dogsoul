# Dogsoul

<img width="1346" height="760" alt="image" src="https://github.com/user-attachments/assets/fbc785ce-14c8-47ee-85cd-4b7dcf6cae07" />


> **Unity**와 **Photon**, **Firebase**를 활용한 멀티플레이 TPS ARPG 프로젝트입니다. 소울류 게임의 메커니즘을 가지되, SD 캐릭터, 빠른 액션, 멀티플레이 협업 요소를 추가하여 캐주얼하게 재해석했습니다.
- **프로젝트 구분**: 팀 프로젝트 (3명)
- **장르**: 멀티플레이 TPS ARPG
- **담당 역할**: 클라이언트 개발(플레이어, 몬스터, 전투), 기획, 팀장
- **개발 환경**: Unity 2022.3.56f1, C#, VSCode, Photon, Firebase
- **개발 기간**: 약 4개월 (2025.01.29~2025.05.16)
- **주요 링크**:
  - 🎬 [YouTube 발표 영상](https://youtu.be/OtntBaEmVDU)
  - 📋 [Google Drive 발표 자료](https://drive.google.com/file/d/1VLTU9QHJnw7QmHsDgh-fGPYmPtici6cO/view?usp=drive_link)

---

## 시스템 구조도
<img width="1350" height="759" alt="image" src="https://github.com/user-attachments/assets/f6c575f1-6a3b-4eee-a377-5caf89066d89" />

---

## 핵심 구현 컨텐츠 및 아키텍처

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

### 2. 거리 기반 AI 및 NavMesh 기반 보스 패턴 파이프라인 구축
- **NavMesh 경로 기반의 정확한 타겟 거리 계산**: NavMeshAgent.CalculatePath로 모퉁이 사이의 실제 이동 경로 거리를 산출하여, 장애물 뒤에 있는 타겟에 대한 비정상적인 추적/공격 판단을 방지했습니다.
- **거리별 다단계 전투 상태 머신(FSM) 구축**: 플레이어와의 거리에 따라 근접 공격, 특수 공격(점프 공격, 투사체 발사), 추적(Chase), 순찰(Patrol), 복귀(BackToSpawn) 상태를 체계적으로 전환하도록 설계했습니다.
- **애니메이션 및 코루틴 기반의 특수 패턴**: 타깃이 멀리 떨어져 있을 때 보스의 점프 공격은 코루틴과 포물선 보간을 통해 역동감 있게 플레이어를 추적할 수 있게 설계했습니다.

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

![인게임](/gifs/jump.gif)


## 트러블슈팅

### 1. 총기 프레임/Notify 의존성으로 인한 연사 속도 오류
- **문제**: 애니메이션 몽타주의 AnimNotify에 격발 로직을 바인딩해 두었으나, 프레임 드랍이나 몽타주 재생 속도 조절 시 실제 프레임에 맞춰 연사 속도가 비정상적으로 빨라지거나 느려지는 현상이 발생했습니다. [YouTube 문제 상황 1](https://www.youtube.com/watch?v=-_sebh7_O0Q)
- **해결**:
  - 무기 데이터(DataTable)에 `FireRate` 항목을 추가하고, `TickComponent` 내에서 DeltaTime 누적 수치(`mTimeSinceLastShot`)를 계산하는 타이머 기반 로직으로 전환했습니다.
  - 애니메이션 재생 타이밍과 실제 격발 타이밍을 분리하여 안정적인 사격 주기를 확보했습니다.

### 2. 높이가 다른 벽 오르기 시 Motion Warping 위치 어긋남
- **문제**: Motion Warping으로 착지/잡기 지점을 지정했으나, 장애물 높이에 따라 캡슐 콜리전과 벽 상단의 연산 지점이 달라지면서 파쿠르 종료 후 플레이어가 공중에 뜨거나 벽 내부로 파묻히는 문제가 있었습니다. [YouTube 문제 상황 2](https://www.youtube.com/watch?v=SwFqHIYX0_8)
- **해결**:
  - Trace로 측정한 실제 벽 높이 오프셋(`mWallHeight`)을 계산하여, 파쿠르 시작 시 플레이어의 캡슐 콜리전 위치와 이동 모드(`MOVE_Flying`)를 수동으로 1차 보정한 뒤 Motion Warping을 수행하도록 수정했습니다.

---

## 개발 회고 및 성찰

- **언리얼 엔진 5 핵심 플러그인 및 시스템 활용**: `EnhancedInput`, `MotionWarping`, `AssetManager` 등 엔진 내장 플러그인과 프레임워크를 프로젝트에 직접 적용하며, 각 기능의 내부 동작 원리와 확장 가능성을 명확히 이해할 수 있었습니다. 언리얼 엔진 개발은 OOP에 대한 깊은 이해도가 필수적이라는 것을 몸소 느낄 수 있었고, 엔진의 잠재능력을 더 끌어올리기 위해서는 C++의 class에 대한 높은 이해도가 있어야 한다는 깨달음을 얻었습니다.
- **데이터 및 이벤트를 통한 구조 개선**: UGameEventMessageSubsystem과 Delegate를 활용한 이벤트 기반 설계를 통해 시스템 간 결합도를 최소화하고, DataTable 기반의 데이터 중심 방식을 적용하여 유지보수성과 확장성이 높고 안정적인 클라이언트 아키텍처의 중요성을 체감했습니다. 팀과 개발하면서 팀원이 내 코드를 유의깊게 볼 수 있고, 내 코드로 컨텐츠를 확장할 수 있다는 점을 깊게 느꼈습니다. 이러한 맥락에서 왜 유지보수성과 가독성이 중요한지 알 수 있었고, 단순히 높은 기술력 뿐만 아니라 팀원의 스타일을 고려하여 코드를 작성할 줄 아는 유연함 역시 개발자에게 필요한 역량이라는 것을 깨달았습니다.
