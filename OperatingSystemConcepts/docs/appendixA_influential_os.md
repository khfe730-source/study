# Appendix A: Influential Operating Systems

현대 OS 설계에 결정적 영향을 미친 역사적 시스템들을 다룬다. 각 시스템이 해결하려 한 문제와 그것이 현대 OS에 남긴 유산을 중심으로 정리한다.

---

## A.1 초기 시스템: 일괄 처리 시대

### IBM 7094 / FMS (Fortran Monitor System)

**1950년대 후반~1960년대 초**

```
운영 방식:
  천공 카드 → Job 덱 제출 → 오퍼레이터가 수동으로 로드 → 실행 → 결과 출력

문제:
- CPU 유휴 시간이 극도로 많음
  (테이프 마운트, 카드 로드 등 I/O 준비 시간 동안 CPU가 놀음)
- 대화형 디버깅 불가
- 하나의 잡이 전체 머신을 독점
```

**기여:** 자원 낭비 문제 의식 → 멀티프로그래밍 필요성 대두

---

## A.2 Atlas (Manchester University, 1959-1962)

**설계자:** Tom Kilburn, 맨체스터 대학

### 혁신적 기여

1. **가상 메모리의 원조**
   ```
   실제 메모리(drum): ~16KB
   가상 주소 공간: ~1MB
   
   Atlas 페이징:
   - 하드웨어 페이지 테이블 (최초!)
   - 페이지 폴트 → 자동 드럼(drum)에서 메모리로 로드
   - LRU 근사 교체 알고리즘
   ```

2. **감시 루틴 (Supervisor) = 현대 커널의 원형**
   - 사용자 프로그램과 특권 코드 분리
   - `extracodes` = 오늘날의 시스템 콜

3. **최초의 스풀링(SPOOLing)**
   - 인쇄 작업을 큐에 쌓아 배치 처리

**현대적 영향:** 가상 메모리, 시스템 콜 인터페이스, 감독자/사용자 모드 분리

---

## A.3 XDS-940 (UC Berkeley, 1967)

### 시분할 시스템 (Time-Sharing)

```
다중 사용자 동시 접속:
  사용자 1 ──┐
  사용자 2 ──┤ → 시분할 → 각자 자기 컴퓨터를 가진 것처럼
  사용자 N ──┘

핵심 혁신:
- 선점형 멀티태스킹 (preemptive multitasking)
- 프로세스별 독립적 메모리 보호
- 재진입 가능(reentrant) 공유 코드
```

**기여:** 현대 대화형 OS의 직접 선조. 다중 사용자 환경 설계 패턴 확립.

---

## A.4 THE (Technische Hogeschool Eindhoven, 1968)

**설계자:** Edsger W. Dijkstra

### 계층적 OS 설계

```
THE 시스템 계층 구조 (하위→상위):
  Level 0: 프로세서 할당 (CPU 스케줄링)
  Level 1: 메모리 관리 (드럼 ↔ 코어 메모리 세그먼트)
  Level 2: 오퍼레이터 콘솔 통신
  Level 3: 입출력 버퍼링
  Level 4: 사용자 프로그램
  Level 5: 오퍼레이터 (인간)

검증: 각 계층은 하위 계층만 사용 → 수학적 정확성 증명 가능
```

**혁신:**
- **세마포어 (Semaphore) 최초 도입** → `P()`, `V()` 연산
- 계층적 모듈 설계 방법론
- OS 정확성을 수학적으로 증명하는 아이디어

**Dijkstra 인용:**
> "The THE multiprogramming system was a grand success, both as a working system and as a body of proof that it was reliable."

**현대적 영향:** 세마포어는 모든 현대 OS 동기화의 기반. 계층형 OS 설계는 마이크로커널, 계층형 I/O 스택으로 이어짐.

---

## A.5 RC 4000 (Regnecentralen, 1969)

**설계자:** Per Brinch Hansen

### 마이크로커널 개념의 원형

```
RC 4000 철학:
"OS는 특정 정책의 구현이 아니라,
 정책을 구현할 수 있는 메커니즘 집합이어야 한다"

핵심: Nucleus (핵)
  - 최소한의 기능만 커널에
  - 프로세스 생성/삭제
  - 메시지 패싱
  - 나머지는 사용자 공간 서버로
```

**혁신:**
- **메시지 패싱 IPC** 최초 체계적 설계
- 정책(policy)과 메커니즘(mechanism) 분리 원칙
- 후일 Mach 마이크로커널, L4, QNX에 직접 영향

---

## A.6 CTSS (Compatible Time-Sharing System, MIT, 1961)

**설계자:** Fernando Corbató (MIT Project MAC)

```
IBM 7094 기반 최초의 실용적 시분할 시스템

혁신:
- 1분 퀀텀 (당시로는 혁명적)
- 파일 시스템 (디렉토리 구조)
- 온라인 에디터
- 최초의 패스워드 인증
- 멀티레벨 피드백 큐 (MLFQ) 스케줄링의 선구

1963: 30명 동시 접속 지원
```

**유산:** MLFQ는 오늘날 Linux, Windows 스케줄러에 직접 이어짐. 파일 시스템 계층 구조 개념.

---

## A.7 Multics (1965-1969)

**배경:** MIT + Bell Labs + GE 공동 프로젝트

### 야심찬 설계 목표

```
"Multiplexed Information and Computing Service"
- 수천 명의 동시 사용자
- 전원 꺼지지 않는 컴퓨터 유틸리티
- 완전한 온라인 파일 저장소
```

### 혁신적 기능들

1. **세그멘테이션 + 페이징 조합**
   ```
   세그먼트: 논리적 단위 (함수, 데이터 구조)
   페이지: 세그먼트를 물리 메모리에 매핑
   → 세그먼트 폴트 → 페이지 단위로 로드
   ```

2. **링 기반 보호 모델**
   ```
   Ring 0: 커널 (최고 권한)
   Ring 1: OS 익스텐션
   Ring 2: 신뢰된 유틸리티
   Ring 3: 일반 사용자
   → Intel x86 보호 링의 직접 원형
   ```

3. **단일 레벨 저장소 (Single-Level Store)**
   ```
   파일 = 메모리 세그먼트
   파일을 열면 주소 공간에 매핑됨 (현대 mmap과 동일)
   ```

4. **ACL (Access Control List)** - 파일 권한의 현대적 모델

### 실패 요인과 교훈

```
- 복잡성 과다: 수백만 줄 코드
- 성능 문제: 혁신 너무 많음 → 실용적 배포 어려움
- GE 635 하드웨어에 종속

Ken Thompson (Bell Labs):
"Multics의 철학에서 배운 것으로 Unix를 만들었다.
 하지만 훨씬 단순하게."
```

**현대적 유산:** 링 기반 보호, mmap, ACL, 계층적 파일 시스템, 동적 링킹

---

## A.8 IBM OS/360 (1964)

**배경:** System/360 아키텍처 출시와 함께 제공

```
OS/360의 목표:
- IBM의 모든 컴퓨터(트랜지스터 → 집적회로)를 하나의 OS로
- 소형(PCP: Primary Control Program)부터 대형(MFT, MVT)까지

PCP: 단일 프로그램 모드
MFT: 고정 파티션 멀티프로그래밍
MVT: 가변 파티션 멀티프로그래밍 (현대 메모리 관리의 원형)
```

**Fred Brooks의 교훈 (Mythical Man-Month):**
```
"OS/360 개발은 내 생애 가장 참담한 관리 경험이었다.
 
 - 5000개 모듈, 100만 줄 코드
 - 지연된 프로젝트에 인력 추가 → 더 늦어짐 (Brooks의 법칙)
 - 소프트웨어 공학(Software Engineering) 탄생의 계기"
```

**기여:** 하드웨어-소프트웨어 아키텍처 분리 (ISA), 대규모 소프트웨어 프로젝트 방법론

---

## A.9 Unix (Bell Labs, 1969-1973)

**설계자:** Ken Thompson, Dennis Ritchie (AT&T Bell Labs)

### 탄생 배경

```
1969: Multics 철수 후 Thompson이 PDP-7에서 개인 프로젝트로 시작
      Space Travel 게임을 실행하기 위한 환경 필요

초기 Unix:
- 어셈블리 작성
- 단일 사용자
- PDP-7 (18비트 워드)

1973: C 언어로 완전 재작성 (Ritchie의 C 개발 후)
→ 이식성 있는 최초의 OS
```

### Unix 철학

```
1. 각 프로그램은 하나의 일을 잘 해라
2. 출력을 다른 프로그램의 입력으로 사용할 수 있게 해라
3. 가능하면 텍스트를 인터페이스로
4. 단순함을 선호하라
```

### 혁신

| 개념 | 설명 | 현대 영향 |
|------|------|-----------|
| 모든 것은 파일 | 장치, 파이프, 소켓 → 파일 인터페이스 | POSIX, Linux |
| 파이프 (`|`) | 프로세스 출력을 다음 입력으로 | 셸 파이프라인 |
| fork/exec | 프로세스 생성의 두 단계 분리 | POSIX 표준 |
| 계층적 파일 시스템 | 단일 루트(`/`)에서 트리 구조 | VFS |
| C 언어로 작성 | 이식 가능한 시스템 프로그래밍 | C 언어 표준화 |

### BSD vs System V 분기

```
1977: Bill Joy (UC Berkeley) → BSD Unix 시작
      → vi, csh, TCP/IP 소켓, 가상 메모리

AT&T System V:
      → SysV IPC (메시지 큐, 세마포어, 공유 메모리)
      → 표준 C 라이브러리

1983: POSIX 표준화 시작 (두 계열 통합)
1991: Linux (Torvalds) - Unix 철학 재구현
```

---

## A.10 CP/M → MS-DOS → Windows

### CP/M (Control Program for Microcomputers, 1974)

```
설계자: Gary Kildall (Digital Research)
플랫폼: Intel 8080/8085 기반

혁신:
- BIOS (Basic Input/Output System) 개념 도입
  → 하드웨어 추상화 계층의 원형
- FAT 파일 시스템의 선조 (CP/M 디렉토리 엔트리)
- 8.3 파일명 규칙

역사적 아이러니:
IBM이 PC OS로 CP/M 대신 MS-DOS를 선택
→ Digital Research의 몰락, Microsoft의 부상
```

### MS-DOS (1981)

```
Microsoft가 Seattle Computer Products의 86-DOS를 구입해 IBM에 라이선스

특징:
- 단일 태스크, 단일 사용자
- COM/EXE 실행 파일 형식
- FAT12/FAT16 파일 시스템
- INT 21h 시스템 콜 인터페이스

한계:
- 1MB 메모리 제한 (8086 20비트 주소)
- 실시간성 없음
- 보호 모드 미사용
```

### Windows 3.x → NT (1985-1993)

```
Windows 1.0/2.x: MS-DOS 위의 협력적 멀티태스킹 GUI 셸

Windows NT 3.1 (1993):
- 완전히 새로운 마이크로커널 기반 설계
- 설계자: Dave Cutler (전 VMS 설계자 @ DEC)
- 선점형 멀티태스킹, 32비트
- POSIX 서브시스템 지원
- NTFS 도입

VMS의 영향:
- VAX/VMS의 프로세스 모델, 보안 모델
- 클러스터형 I/O 스택
- 페이지 파일 관리
```

---

## A.11 Mach (Carnegie Mellon University, 1985-1994)

*(상세 내용은 Appendix C 참고)*

```
핵심 혁신:
- 마이크로커널 아키텍처의 실용적 구현
- 포트(Port)와 메시지(Message) IPC
- 가상 메모리 외부화 (memory objects)
- 멀티프로세서 지원

영향:
- macOS/iOS의 커널 (XNU = Mach + BSD)
- GNU Hurd
- OSF/1 (DEC Alpha)
```

---

## A.12 주요 OS 혁신 연대표

```
1959  Atlas: 가상 메모리, 감독자 모드
1961  CTSS: 시분할, MLFQ
1965  Multics: 링 보호, ACL, mmap, 계층적 FS
1968  THE: 세마포어, 계층적 설계
1969  RC 4000: 마이크로커널/메시지 패싱 원형
1969  Unix: 이식 가능한 OS, 파이프, fork/exec
1974  CP/M: BIOS, 개인용 OS
1981  MS-DOS: 대중화된 개인 OS
1985  Mach: 실용적 마이크로커널
1987  Minix: 교육용, Linux의 직접 영감
1991  Linux: 오픈소스 Unix 클론
1993  Windows NT: 기업용 마이크로커널 하이브리드

현대 OS 계보:
Unix ──→ BSD ──→ macOS (XNU = Mach + BSD)
     └──→ Linux
     └──→ Solaris

Windows:
VMS ──→ Windows NT ──→ Windows 10/11
```

---

## 요약

```
현대 OS 핵심 개념의 기원:
┌────────────────────────────────────────────────────────┐
│ 가상 메모리      → Atlas (1959)                        │
│ 시분할           → CTSS (1961)                         │
│ 보호 링          → Multics (1965)                      │
│ 세마포어         → THE (1968)                          │
│ 마이크로커널/IPC → RC 4000 (1969)                      │
│ 파이프/fork/exec → Unix (1969)                         │
│ BIOS 추상화      → CP/M (1974)                         │
│ 실용적 마이크로커널 → Mach (1985)                      │
│ 오픈소스 Unix    → Linux (1991)                        │
│ 기업형 NT 설계   → Windows NT (1993)                   │
└────────────────────────────────────────────────────────┘
```
