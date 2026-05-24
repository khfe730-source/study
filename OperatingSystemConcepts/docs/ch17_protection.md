# Chapter 17: Protection

## 보안(Security) vs 보호(Protection)

- **보안(Security)**: 외부 위협으로부터 시스템 전체를 지키는 것 (Ch16)
- **보호(Protection)**: OS 내부에서 프로세스·사용자의 자원 접근을 제어하는 **메커니즘**

---

## 17.1 보호의 목표 (Goals of Protection)

보호가 필요한 이유:
1. **의도적 위반 방지**: 접근 제한을 고의로 우회하는 사용자 차단
2. **정책 준수 보장**: 모든 프로세스가 선언된 정책에 맞게 자원 사용
3. **신뢰성 향상**: 서브시스템 간 인터페이스 오류를 조기 탐지 → 결함 확산 방지

**핵심 원칙: 정책과 메커니즘의 분리 (Separation of Policy and Mechanism)**
- **메커니즘(Mechanism)**: 어떻게 할 것인가 (OS가 제공)
- **정책(Policy)**: 무엇을 할 것인가 (사용자·관리자·앱이 결정)
- 분리의 이유: 정책은 시간·장소에 따라 변하지만 메커니즘은 안정적이어야 함

---

## 17.2 보호 원칙 (Principles of Protection)

### 최소 권한 원칙 (Principle of Least Privilege)

> "프로그램, 사용자, 시스템은 작업 수행에 필요한 최소 권한만 가져야 한다."

```
잘못된 예: 항상 root로 실행
  → 악성 코드가 root 권한 획득 시 시스템 전체 장악

올바른 예: 필요한 작업에만 권한 부여
  → 공격 성공해도 피해 범위가 해당 권한 내로 제한
  → OS 수준의 면역 시스템 역할
```

### 구획화 (Compartmentalization)

- 각 시스템 컴포넌트를 특정 권한과 접근 제한으로 보호
- 하나의 컴포넌트가 침해되어도 다음 방어선 유지
- 구현 형태: 네트워크 DMZ, 가상화, 컨테이너

### 심층 방어 (Defense in Depth)

- 단일 원칙으로는 모든 취약점 해결 불가
- 여러 층의 보호를 중첩 적용 (성(castle): 수비대 + 성벽 + 해자)

---

## 17.3 보호 링 (Protection Rings)

### Bell-LaPadula 모델 기반 링 구조

```
Ring 0 (최고 권한, 커널)
  ├── Ring 1
  │    ├── Ring 2
  │    │    └── Ring N-1 (최저 권한, 사용자)
  
규칙: Ring i는 Ring j (j < i)의 기능 부분 집합
높은 권한 링 진입: gate 명령 (syscall 등) 통해서만 가능
```

**부팅 시**: 가장 높은 권한 레벨에서 시작 → 초기화 후 낮은 레벨로 전환

### Intel 아키텍처

```
Ring -1: 하이퍼바이저(VMM) — 가상화 전용
Ring 0:  커널 (kernel mode)
Ring 1-2: OS 서비스 (일반적으로 사용 안 함)
Ring 3:  사용자 프로세스 (user mode)

EFLAGS 레지스터의 2비트로 모드 구분
Ring 3에서 EFLAGS 접근 불가 → 권한 상승 차단
```

### ARM 아키텍처

```
ARMv7 (TrustZone 포함):
  USR 모드  → 사용자 공간
  SVC 모드  → 커널/수퍼바이저 모드
  TrustZone → 가장 높은 권한; 하드웨어 암호화 키 독점 접근

  SMC (Secure Monitor Call): 커널에서만 TrustZone 진입 가능
  TrustZone 기능: NFC Secure Element, 온칩 암호화 키
  → 커널이 침해되어도 키 직접 탈취 불가

ARMv8 예외 레벨(Exception Levels):
  EL0: 사용자 모드
  EL1: 커널 모드
  EL2: 하이퍼바이저
  EL3: 보안 모니터 (TrustZone, 최고 권한)

Android 5.0+: TrustZone 적극 활용
Samsung RKP, Apple KPP(WatchTower): EL3에서 커널 무결성 검사
```

---

## 17.4 보호 도메인 (Domain of Protection)

### 접근 권리 (Access Rights)

```
시스템 = 프로세스 + 객체의 집합

객체(Object):
  하드웨어: CPU, 메모리 세그먼트, 프린터, 디스크
  소프트웨어: 파일, 프로그램, 세마포어

접근 권리(Access Right) = <object-name, rights-set>
  예: <파일 F, {read, write}>

도메인(Domain) = 접근 권리의 집합
  프로세스는 하나의 도메인에서 실행
  실행 가능한 연산 = 해당 도메인의 접근 권리 집합
```

**필요 지식 원칙(Need-to-Know Principle):**
- 프로세스는 현재 작업에 필요한 객체만 접근 가능해야 함
- 컴파일러 호출 시: 소스 파일, 출력 오브젝트 파일에만 접근 (다른 파일 접근 불가)
- Need-to-Know = 정책, Least Privilege = 메커니즘

### 도메인 구조 (Domain Structure)

```
도메인 예시 (3개 도메인):

       D1             D2              D3
  <O1, {r,w}>   <O2, {w}>       <O1, {exec}>
  <O2, {exec}>  <O4, {print}>   <O3, {r}>
  <O3, {r,w}>                   <O4, {print}>

  도메인 공유: <O4, {print}>은 D2, D3 공유

도메인 구현 방식:
  ① 사용자 = 도메인: 사용자 교체 시 도메인 전환
  ② 프로세스 = 도메인: 프로세스 간 메시지 전달 시 전환
  ③ 프로시저 = 도메인: 프로시저 호출 시 전환
```

**정적 vs 동적 도메인:**
- **정적(Static)**: 프로세스 생애 동안 자원 집합 고정; 필요 지식 원칙 위반 가능성
- **동적(Dynamic)**: 도메인 전환(switching) 허용; 최소 권한 정밀 적용 가능

### 17.4.2 예시: UNIX의 setuid

```
setuid 비트 (chmod +s):
  실행 시 파일 소유자의 ID로 임시 실행

예시: passwd 명령
  소유자: root / setuid 활성화
  실행 시: user → root 도메인으로 일시 전환
  동작: 현재 사용자 패스워드만 수정 가능 (검증 후 제한적 접근)

위험성:
  setuid 바이너리에 버퍼 오버플로우 → 즉각적 root 권한 획득
  → 최소 권한 원칙 엄수 필요
```

### 17.4.3 예시: Android 앱 ID

```
설치 시: installd 데몬이 각 앱에 고유 UID/GID 할당
데이터 디렉토리: /data/data/<app-name> (해당 UID/GID 소유)

격리 메커니즘:
  • 앱 간: UNIX 사용자 격리와 동일 수준
  • 네트워킹: AID_INET(3003) GID만 소켓 허용
  • isolated UID: 대부분 서비스에 RPC 요청 불가 (최소 권한)
```

---

## 17.5 접근 행렬 (Access Matrix)

**접근 행렬**: 보호의 일반 추상 모델

```
         F1       F2       F3     Printer
D1    read              read
D2              print
D3    read    write    execute
D4    read    write    read
      write            write

행(row) = 도메인
열(column) = 객체
entry access(i,j) = Di가 Oj에서 수행 가능한 연산 집합

도메인 전환 제어 (도메인을 객체로 포함):
      F1    D1      D2      D3      D4     Printer
D1         switch
D2                 switch  switch
D3         switch
D4   ...

access(i,j)에 switch ∈ 포함 시 Di → Dj 전환 허용
```

### 동적 접근 행렬 제어 연산

| 연산 | 의미 |
|------|------|
| **copy (*표시)** | access(i,j)의 권리를 동일 열의 다른 항목으로 복사 |
| **transfer** | 복사 후 원래 항목에서 삭제 |
| **limited copy** | 복사된 권리에는 * 없음 (추가 전파 불가) |
| **owner** | access(i,j)에 owner 포함 시 Di는 열 j의 모든 항목 추가/삭제 가능 |
| **control** | access(i,j)에 control 포함 시 Di는 도메인 Dj의 행 항목 삭제 가능 |

**감금 문제(Confinement Problem)**: 특정 도메인에 있던 정보가 실행 환경 밖으로 유출되지 않음을 보장하는 것 — 일반적으로 해결 불가

---

## 17.6 접근 행렬 구현 (Implementation of the Access Matrix)

접근 행렬은 대부분의 항목이 비어있는 **희소 행렬(sparse matrix)**

### 17.6.1 전역 테이블 (Global Table)

- `<domain, object, rights-set>` 삼중쌍 목록
- 접근 시 테이블 탐색
- **단점**: 테이블이 크면 메모리 외 보관 → 추가 I/O 필요; 그룹화 활용 어려움

### 17.6.2 객체별 접근 목록 (Access Lists for Objects)

```
각 객체(열)마다 비어있지 않은 항목만 보관:
  파일 F: [<D1, {read}>, <D3, {read,write}>, <D4, {read,write}>]
  기본 접근권(default set): 미등록 도메인에 적용

장점: 사용자 관점에 직관적 (파일 생성 시 도메인별 권한 직접 지정)
단점: 특정 도메인의 모든 권한 파악 어려움; 매 접근마다 탐색 필요
```

### 17.6.3 도메인별 캐퍼빌리티 목록 (Capability Lists for Domains)

```
각 도메인(행)마다 접근 가능 객체 목록:
  D1: [파일 F1 (read), 파일 F3 (read)]
  D4: [파일 F1 (read,write), 파일 F3 (read,write), ...]

캐퍼빌리티(Capability) = 물리적 이름/주소로 된 객체 참조
  • 소유 = 접근 허용
  • 직접 접근 불가: OS가 관리, 프로세스는 간접 접근만 가능
  • 태그(Tag) 또는 주소 공간 분리로 변조 방지

장점: 도메인별 권한 정보 국소화
단점: 권리 철회(revocation) 어려움
```

### 17.6.4 Lock-Key 메커니즘

- 객체마다 **잠금(locks)** 목록, 도메인마다 **키(keys)** 목록
- 키가 잠금과 일치할 때만 접근 허용
- 접근 목록과 캐퍼빌리티의 절충안

### 17.6.5 비교

| 방법 | 장점 | 단점 |
|------|------|------|
| 전역 테이블 | 단순 | 크기 큼, 그룹화 어려움 |
| 접근 목록 | 사용자 직관적; 권한 즉시 철회 | 매 접근마다 탐색; 도메인별 권한 파악 어려움 |
| 캐퍼빌리티 목록 | 도메인 권한 국소화; 검증 빠름 | 철회 어려움; 사용자 비직관적 |
| Lock-Key | 유연, 효과적 | 키 관리 필요 |

**실제 UNIX 방식 (혼합 전략):**
1. `open()` 시 접근 목록 탐색 → 허용 시 파일 테이블에 캐퍼빌리티 항목 생성
2. 이후 모든 접근은 캐퍼빌리티(파일 디스크립터) 확인으로 빠르게 처리
3. `close()` 시 캐퍼빌리티 삭제

---

## 17.7 접근 권리 철회 (Revocation of Access Rights)

| 구분 | 종류 |
|------|------|
| **시점** | 즉시(Immediate) vs 지연(Delayed) |
| **범위** | 전체(General) vs 선택적(Selective) |
| **정도** | 전부(Total) vs 부분(Partial) |
| **기간** | 영구(Permanent) vs 임시(Temporary) |

**접근 목록 방식**: 목록에서 삭제 → 즉시·선택적·부분적·영구/임시 가능 (쉬움)

**캐퍼빌리티 방식의 철회 기법:**

```
1. 재획득(Reacquisition):
   주기적으로 캐퍼빌리티 삭제 → 프로세스가 재획득 시도 → 철회된 경우 재획득 실패

2. 역방향 포인터(Back-Pointers):
   각 객체에 관련 캐퍼빌리티 포인터 목록 유지
   → 철회 시 포인터 추적하여 캐퍼빌리티 변경 (MULTICS 방식)
   → 구현 비용 높음

3. 간접 참조(Indirection):
   캐퍼빌리티 → 전역 테이블 항목 → 객체
   → 철회: 테이블 항목 삭제 → 캐퍼빌리티 무효화 (CAL 방식)
   → 선택적 철회 어려움

4. 키(Keys):
   각 객체에 마스터 키 연결; 캐퍼빌리티 생성 시 현재 마스터 키 복사
   → 철회: set-key로 마스터 키 교체 → 이전 캐퍼빌리티 모두 무효
   → 키 목록으로 선택적 철회 가능
```

---

## 17.8 역할 기반 접근 제어 (Role-Based Access Control, RBAC)

Solaris 10+에서 최소 권한 원칙을 명시적으로 구현:

```
권한(Privilege): 시스템 콜 실행 또는 옵션 사용 권한
역할(Role): 권한들의 집합
사용자: 역할 배정 또는 패스워드로 역할 획득

흐름:
  User 1 → Role 1 → Privileges 1 → Process (Role 1 권한으로 실행)

장점:
  • superuser/setuid 프로그램 관련 보안 위험 감소
  • 접근 행렬의 구체적 구현 (도메인=역할, 객체=권한)
```

---

## 17.9 강제 접근 제어 (Mandatory Access Control, MAC)

### DAC vs MAC

| 구분 | DAC (Discretionary AC) | MAC (Mandatory AC) |
|------|----------------------|--------------------|
| 제어 주체 | 자원 소유자 | 시스템 정책 |
| root 한계 | root가 모든 권한 | root도 MAC 정책 변경 불가 |
| 취약점 | root 침해 시 모든 방어 무너짐 | 침해된 root도 정책 우회 불가 |

**MAC 구현 방식:**

```
레이블(Label): 객체(파일, 장치)와 주체(프로세스)에 부여
정책: 주체 레이블이 객체 레이블에 접근 가능한지 결정

예시 레이블 순서: unclassified → secret → top secret

"secret" 프로세스:
  → unclassified, secret 파일 접근 가능
  → top secret 파일: OS가 존재 자체를 숨김 (디렉토리 목록에서 제외)
  → unclassified 프로세스로의 IPC: 차단
```

**주요 MAC 구현:**

| OS | 구현 |
|----|------|
| Linux | **SELinux** (NSA 개발, 대부분 배포판 통합) |
| macOS | **TrustedBSD** (FreeBSD 5.0 → macOS 10.5 채택) |
| iOS | TrustedBSD 기반, 대부분 보안 기능의 기반 |
| Windows | **Mandatory Integrity Control** (Vista+) |
| Solaris | **Trusted Solaris 2.5** (최초 MAC 도입) |

---

## 17.10 캐퍼빌리티 기반 시스템 (Capability-Based Systems)

### 17.10.1 Linux 캐퍼빌리티 (POSIX.1e 기반)

```
root 권한을 비트맵으로 세분화:

비트마스크 (permitted/effective/inheritable):

  CAP_NET_BIND_SERVICE  → 1024 미만 포트 바인딩
  CAP_SYS_PTRACE        → 프로세스 추적
  CAP_KILL              → 임의 프로세스에 시그널 전송
  CAP_SYS_ADMIN         → 다양한 관리 작업
  ... (총 수십 개)

사용 패턴:
  프로세스 시작: 전체 permitted 캐퍼빌리티 집합 보유
  실행 중: 필요 없어진 캐퍼빌리티 자발적 반납
  한 번 반납된 캐퍼빌리티: 재획득 불가

예: 네트워크 포트 개방 후 CAP_NET_BIND_SERVICE 제거
  → 이후 추가 포트 개방 불가 (침해 시 피해 최소화)

한계:
  • 비트맵 방식 → 동적 추가 불가 (커널 재컴파일 필요)
  • 커널 레벨 캐퍼빌리티에만 적용
```

### 17.10.2 Darwin 엔타이틀먼트 (Apple)

```xml
<!-- Darwin 엔타이틀먼트 예시 (XML plist) -->
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" ...>
<plist version="1.0">
<dict>
  <key>com.apple.private.kernel.get-kext-info</key>
  <true/>
  <key>com.apple.rootless.kext-management</key>
  <true/>
</dict>
</plist>
```

- 프로그램이 필요한 권한을 **선언(declarative)** 형태로 명시
- 코드 서명(code signature)에 엔타이틀먼트 내장 → 위조 불가
- 검증: 코드 서명에서 엔타이틀먼트 조회 → 문자열 매칭
- 시스템 엔타이틀먼트(`com.apple.*`): Apple 자체 바이너리만 사용 가능

---

## 17.11 기타 보호 개선 방법 (Other Protection Improvement Methods)

### 17.11.1 시스템 무결성 보호 (System Integrity Protection, SIP)

macOS 10.11+에서 도입:

```
목표: root도 시스템 파일 수정 불가

구현:
  • 파일 확장 속성으로 restricted 마킹
  • 시스템 바이너리 디버깅/검사 차단
  • 코드 서명된 커널 확장(kext)만 허용

root가 여전히 할 수 있는 것: 사용자 파일 관리, 앱 설치
root가 할 수 없는 것: OS 컴포넌트 교체·수정

구현 메커니즘: 모든 프로세스에 적용되는 글로벌 정책
  예외: fsck, kextload 등 특별히 entitled된 시스템 바이너리만 허용
```

### 17.11.2 시스템 콜 필터링 (System-Call Filtering)

```
목표: 프로세스별 허용 시스템 콜 집합 제한

Linux SECCOMP-BPF:
  • Berkeley Packet Filter 언어로 커스텀 프로파일 정의
  • prctl() 시스템 콜로 프로파일 로드
  • main() 실행 전 런타임 라이브러리 또는 로더에서 적용

심층 필터링:
  • 시스템 콜 인수(argument)까지 검사
  • 예: futex 시스템 콜의 race condition 취약점 → 특정 인수 차단

모듈식 구현 (유연성):
  커널 callout → 외부 드라이버/모듈/확장에서 필터링 로직 제공
  → 커널 재컴파일 없이 필터 업데이트 가능
  → 전문 프로파일 언어 + 인터프리터 내장
```

### 17.11.3 샌드박싱 (Sandboxing)

```
개념: 프로세스를 허용된 작업만 할 수 있는 환경에 격리

적용 시점: fork() 직후, main() 실행 전
메커니즘: 제거 불가능한 제한 집합 적용

구현 사례:
  • Java/.NET: 가상 머신 레벨에서 샌드박스
  • Android: SELinux 정책 + SECCOMP-BPF
    - SELinux: 시스템 속성·서비스 엔드포인트 레이블링
    - SECCOMP-BPF: Bionic(C runtime)이 모든 앱에 시스템 콜 제한 적용
  • Apple macOS/iOS:
    - macOS 10.5 "Seatbelt": opt-in, Scheme 언어 기반 동적 프로파일
    - iOS: 필수(mandatory), 코드 서명과 함께 미신뢰 코드 차단 주요 수단
    - macOS 10.8+: Mac Store 앱에 필수 적용
    - SIP(10.11+): 시스템 전체 플랫폼 프로파일로 진화
```

**샌드박스 프로파일 예시 (macOS 데몬):**

```scheme
(version 1)
(deny default)              ; 모든 작업 기본 차단
(allow file-chroot)
(allow file-read-metadata (literal "/var"))
(allow sysctl-read)
(allow mach-per-user-lookup)
(allow mach-lookup)
  (global-name "com.apple.system.logger")
```

### 17.11.4 코드 서명 (Code Signing)

```
목적: 프로그램이 작성자 생성 이후 변경되지 않음을 확인

기법: 암호화 해시 + 디지털 서명
      해시(프로그램) → 개인키로 서명 → 공개키로 검증

활용 사례:
  OS 배포판, 패치, 서드파티 도구 서명
  Apple: 구 iOS 버전 앱 서명 중단 → App Store 다운로드 시 실행 불가
  iOS/Windows/macOS: 서명 검증 실패 시 실행 거부

Apple 엔타이틀먼트와의 통합:
  엔타이틀먼트 → 코드 서명에 내장 → 변조 불가
```

---

## 17.12 언어 기반 보호 (Language-Based Protection)

### 17.12.1 컴파일러 기반 강제 (Compiler-Based Enforcement)

보호를 언어 타입 시스템의 확장으로 표현:

```
장점:
  1. 보호 요구사항을 OS 시스템 콜 대신 선언적(declarative)으로 명시
  2. 특정 OS 시설에 독립적 표현 가능
  3. 서브시스템 설계자가 강제 메커니즘을 직접 제공할 필요 없음
  4. 접근 권한이 데이터 타입 개념과 자연스럽게 결합

컴파일러 역할:
  • 보호 위반 불가 참조 → 검사 없이 통과
  • 위반 가능 참조 → 런타임 검사 코드 삽입
  • 정적 검사로 실행 시간 오버헤드 최소화

보안 특성 비교:
  커널 강제: 높은 보안성 (하드웨어 기반)
  컴파일러 강제: 유연성·효율성 높음 (off-line 컴파일 시 정적 검사)
```

### 17.12.2 Java 스택 검사 (Stack Inspection)

```
JVM 내 보호 도메인:
  클래스 로드 시 URL + 디지털 서명 → 보호 도메인 할당
  정책 파일: 도메인별 권한 설정
    신뢰 서버 클래스 → 사용자 홈 디렉토리 접근 허용
    비신뢰 서버 클래스 → 파일 접근 권한 없음

스택 검사 메커니즘:
  doPrivileged() 블록: 스택 프레임에 어노테이션 삽입
  checkPermissions(): 스택을 최신→최초 순서로 검사

checkPermissions() 판단 규칙:
  ① doPrivileged() 어노테이션 발견 → 즉시 허용 반환
  ② 해당 도메인에서 접근 거부 → AccessControlException 발생
  ③ 스택 소진 → 구현에 따라 허용 또는 거부
```

```
스택 검사 예시:
  [gui() - 비신뢰 앱 도메인]
       ↓ get(url) 호출
  [get() - URL 로더 도메인 (*.lucent.com:80 소켓 허용)]
       ↓ doPrivileged { open('proxy.lucent.com:80') }
  [networking open() - checkPermissions()]
       → doPrivileged 어노테이션 발견 → 허용 ✓

  [gui() - 비신뢰 앱 도메인]
       ↓ open(addr) 직접 호출
  [networking open() - checkPermissions()]
       → gui() 프레임에서 접근 거부 발견 → 예외 발생 ✗
```

**Java 타입 안전성(Type Safety):**
- 프로그램이 메모리에 직접 접근 불가 (참조를 통한 간접 접근만 허용)
- 참조 위조 불가 → 런타임 스택 조작 불가
- `private` / `protected` 접근 제어: 타입 안전성으로 강제
- 로드 시간 + 런타임 검사로 타입 안전성 보장 → 보호의 근간

---

## 핵심 비교 요약

### 접근 제어 모델 비교

| 모델 | 특징 | 대표 구현 |
|------|------|---------|
| **DAC** | 소유자가 권한 설정 | UNIX rwx, Windows ACL |
| **RBAC** | 역할 단위 권한 관리 | Solaris 10, Linux sudo |
| **MAC** | 시스템 정책으로 강제 | SELinux, macOS TrustedBSD |
| **캐퍼빌리티** | 소유 = 접근 허용 | Linux capabilities, Darwin entitlements |

### 보호 링 구현 비교

```
Intel:
  Ring -1 (Hypervisor) > Ring 0 (Kernel) > Ring 3 (User)

ARMv8:
  EL3 (Secure Monitor) > EL2 (Hypervisor) > EL1 (Kernel) > EL0 (User)
  
TrustZone의 특수성:
  커널(EL1)에서 TrustZone(EL3)로의 진입: SMC 명령 필요
  TrustZone의 암호화 키: 커널에서도 직접 접근 불가
```

### 접근 행렬 구현 선택 가이드

```
주된 사용 패턴:
  객체별 접근 제어 → 접근 목록(ACL)
  프로세스별 권한 파악 → 캐퍼빌리티 목록
  빠른 철회 필요 → 접근 목록 (즉시 삭제)
  빠른 접근 검증 → 캐퍼빌리티 (소유 = 허용)
  대부분의 실제 시스템 → 혼합 방식 (open 시 ACL, 이후 capability)
```
