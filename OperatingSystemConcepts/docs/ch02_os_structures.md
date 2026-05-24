# Chapter 2: Operating-System Structures

> **핵심 한 줄 요약**: OS의 구조는 서비스 제공 → 시스템 콜 인터페이스 → 내부 설계(모놀리식/마이크로커널/모듈/하이브리드)의 세 관점으로 분석하며, 메커니즘(mechanism)과 정책(policy)의 분리가 설계의 핵심 원칙이다.

---

## 목차

- [2.1 운영체제 서비스](#21-운영체제-서비스)
- [2.2 사용자-OS 인터페이스](#22-사용자-os-인터페이스)
- [2.3 시스템 콜](#23-시스템-콜)
- [2.4 시스템 서비스](#24-시스템-서비스)
- [2.5 링커와 로더](#25-링커와-로더)
- [2.6 애플리케이션이 OS 종속적인 이유](#26-애플리케이션이-os-종속적인-이유)
- [2.7 OS 설계와 구현](#27-os-설계와-구현)
- [2.8 운영체제 구조](#28-운영체제-구조)
- [2.9 OS 빌드와 부팅](#29-os-빌드와-부팅)
- [2.10 운영체제 디버깅](#210-운영체제-디버깅)

---

## 2.1 운영체제 서비스

OS는 프로그램의 실행 환경을 제공하며, 두 그룹의 서비스를 제공한다.

### 사용자를 위한 서비스

```
┌─────────────────────────────────────────────────────────────┐
│          user and other system programs                      │
├──────────────────────────┬──────────────────────────────────┤
│      User Interface      │    OS Services for Users         │
│   (GUI / CLI / Touch)    │  - Program execution             │
│                          │  - I/O operations                │
│     System Calls ───────────────────────────────            │
│                          │  - File-system manipulation      │
├──────────────────────────┤  - Communications               │
│    Operating System      │  - Error detection               │
└──────────────────────────┴──────────────────────────────────┘
```

| 서비스 | 설명 |
|--------|------|
| **User interface** | GUI, CLI, 터치스크린 — 사용자 진입점 |
| **Program execution** | 프로그램 메모리 로드 및 실행 (정상/비정상 종료 포함) |
| **I/O operations** | 효율성/보호를 위해 사용자는 직접 I/O 제어 불가 → OS 경유 |
| **File-system manipulation** | 파일/디렉터리 생성·삭제·읽기·쓰기·권한 관리 |
| **Communications** | 같은 컴퓨터 또는 네트워크 상의 프로세스 간 정보 교환. 공유 메모리(shared memory) 또는 메시지 패싱(message passing) 방식 |
| **Error detection** | CPU/메모리 오류, I/O 장치 오류, 사용자 프로그램 오류 지속 감지 및 대응 |

### 시스템 효율을 위한 서비스

| 서비스 | 설명 |
|--------|------|
| **Resource allocation** | 멀티프로세스 환경에서 CPU, 메모리, 파일 저장소, I/O 장치 할당 |
| **Logging** | 자원 사용 통계 추적 — 과금(accounting) 또는 성능 분석에 활용 |
| **Protection and security** | 자원 접근 제어, 사용자 인증(authentication), 외부 공격 방어 |

---

## 2.2 사용자-OS 인터페이스

### 2.2.1 명령 인터프리터 (Command Interpreter / Shell)

OS가 사용자에게 제공하는 텍스트 기반 인터페이스. 로그인 시 또는 프로세스 시작 시 실행.

- UNIX/Linux: bash, csh, ksh, zsh 등 **셸(shell)**이라 부름
- 명령 구현 방식:
  1. **인터프리터가 직접 포함**: 명령 수가 늘수록 인터프리터 크기 증가
  2. **외부 프로그램 방식 (UNIX 방식)**: 명령 = 파일 이름. 인터프리터는 해당 파일을 찾아 로드·실행. `rm file.txt` → `rm`이라는 파일을 찾아 `file.txt`를 인자로 실행. 새 명령 추가가 쉽고 인터프리터 크기가 고정됨

**셸 스크립트(shell script)**: CLI 명령 시퀀스를 파일에 기록하여 프로그램처럼 실행. 컴파일 없이 인터프리터가 해석.

### 2.2.2 그래픽 사용자 인터페이스 (GUI)

마우스/트랙패드 기반의 데스크톱 메타포(desktop metaphor). Xerox PARC(1970년대) → Apple Macintosh(1980년대) → 광범위 보급.

- macOS: **Aqua** (마우스/트랙패드 기반)
- Linux: **KDE**, **GNOME** (오픈소스)
- Windows: GUI + 셸 인터페이스 병행

### 2.2.3 터치스크린 인터페이스

모바일 디바이스에 적합. 물리적 키보드 없이 터치/스와이프 제스처로 상호작용.
- iOS: **Springboard**
- 가상 키보드, 음성 인식(Siri) 등 보완 입력 방식 병행

### 2.2.4 인터페이스 선택

- **CLI 선호**: 시스템 관리자, 파워 유저 — 반복 작업의 스크립팅, 더 빠른 접근
- **GUI 선호**: 일반 사용자 — 직관성
- UI는 OS 구조와 실질적으로 독립적임 (OS 설계의 직접 기능이 아님)

---

## 2.3 시스템 콜 (System Calls)

OS가 제공하는 서비스에 대한 프로그래밍 인터페이스. C/C++로 제공되며, 일부 저수준 작업은 어셈블리 필요.

### 2.3.1 시스템 콜 사용 예: 파일 복사

간단한 파일 복사 프로그램조차 수십 개의 시스템 콜을 사용한다:

```
1. 입력 파일명 획득 → write(prompt) + read(keyboard)
2. 출력 파일명 획득 → write(prompt) + read(keyboard)
3. 입력 파일 open() → 오류 시 write(error) + exit()
4. 출력 파일 create()/open() → 동명 파일 존재 시 처리
5. loop:
   read(input_file) → write(output_file) until EOF
6. close(output_file)
7. write(completion_msg)
8. exit(0)  ← 정상 종료
```

### 2.3.2 응용 프로그래밍 인터페이스 (API)

대부분의 프로그래머는 시스템 콜을 직접 호출하지 않고 **API**를 통해 접근한다.

```
User Application
     │
     │  API 호출 (예: read())
     ▼
┌─────────────────┐
│   System-Call   │  ← RTE가 관리. 함수 번호 테이블로 인덱싱
│   Interface     │
└────────┬────────┘
         │  실제 시스템 콜 호출 (번호 기반)
         ▼
┌─────────────────┐
│   OS Kernel     │  ← 시스템 콜 구현 실행
│  (kernel mode)  │
└─────────────────┘
```

주요 API:
- **Windows API**: Windows 시스템용. `CreateProcess()` → 내부적으로 `NTCreateProcess()` 커널 콜
- **POSIX API**: UNIX, Linux, macOS. `libc` 라이브러리 경유. `read()`, `write()`, `fork()` 등
- **Java API**: JVM 위에서 실행

**RTE (Run-Time Environment)**: 컴파일러/인터프리터 + 라이브러리 + 로더를 포함한 실행 환경 전체. 시스템 콜 인터페이스를 관리한다.

**파라미터 전달 방식** (3가지):
1. **레지스터**: 파라미터를 직접 레지스터에 전달 (가장 단순, 파라미터 수 제한)
2. **메모리 블록/테이블**: 파라미터를 메모리에 저장 후 주소를 레지스터로 전달 (Linux: 파라미터 > 5개일 때)
3. **스택**: 프로그램이 스택에 push → OS가 pop

```c
// POSIX read() API 예시
#include <unistd.h>
ssize_t read(int fd, void *buf, size_t count);
// fd: 파일 디스크립터
// buf: 데이터를 읽어들일 버퍼
// count: 최대 읽을 바이트 수
// 반환: 읽은 바이트 수, 0 (EOF), -1 (에러)
```

### 2.3.3 시스템 콜의 종류

6개 카테고리로 분류:

#### 1. 프로세스 제어 (Process Control)

| 시스템 콜 | Windows | UNIX/Linux |
|----------|---------|-----------|
| 프로세스 생성 | `CreateProcess()` | `fork()` |
| 프로세스 종료 | `ExitProcess()` | `exit()` |
| 대기 | `WaitForSingleObject()` | `wait()` |
| 프로그램 로드/실행 | — | `exec()` |
| 잠금 | `AcquireSRWLockExclusive()` | `acquire_lock()` |

**FreeBSD 멀티태스킹 예**:
```
shell → fork() → 새 프로세스 생성
             └→ exec("program") → 프로그램 로드 및 실행
             └→ exit(status) → 종료 시 부모 프로세스에 상태 반환
```

**Arduino 단일태스킹 예**: OS 없음. boot loader가 sketch를 flash에 로드 → 단 하나의 sketch만 실행.

#### 2. 파일 관리 (File Management)

`create()`, `delete()`, `open()`, `close()`, `read()`, `write()`, `reposition()`, `get/set_file_attributes()`

#### 3. 장치 관리 (Device Management)

물리적 장치(디스크)와 추상적 장치(파일) 모두 포함. UNIX는 파일과 장치를 통합된 구조로 처리.

`request()`, `release()`, `read()`, `write()`, `reposition()`, `get/set_device_attributes()`

#### 4. 정보 유지 관리 (Information Maintenance)

`time()`, `date()`, `dump()` (디버깅용), `get/set_process_attributes()`

Linux `strace`: 프로세스가 실행하는 모든 시스템 콜을 순서대로 출력.

#### 5. 통신 (Communications)

**메시지 패싱 모델**:
```
Process A ──→ open_connection() ──→ send/receive_message() ──→ close_connection() ──→ Process B
```
- 소량 데이터 교환에 적합. 컴퓨터 간 통신에 구현이 쉬움.
- 클라이언트-서버 모델. 수신측은 보통 데몬(daemon) 프로세스.

**공유 메모리 모델**:
```
Process A ─┐
           ├─→ shared_memory_create/attach() → [공유 메모리 영역]
Process B ─┘
```
- 메모리 전송 속도로 대용량 통신 가능. 단, 보호와 동기화 문제 존재.

#### 6. 보호 (Protection)

`set_permission()`, `get_permission()`, `allow_user()`, `deny_user()`

---

## 2.4 시스템 서비스 (System Services)

시스템 유틸리티(system utilities)라고도 불리며, 프로그램 개발과 실행을 위한 편리한 환경을 제공한다.

| 카테고리 | 예시 |
|---------|------|
| **파일 관리** | cp, mv, rm, ls, mkdir |
| **상태 정보** | date, df, du, top, ps (일부는 레지스트리 활용) |
| **파일 수정** | 텍스트 에디터 (vim, emacs, nano) |
| **프로그래밍 언어 지원** | GCC, Python 인터프리터, JDK |
| **프로그램 로딩/실행** | 링커, 로더, 디버거 (gdb) |
| **통신** | ssh, ftp, email 클라이언트, 웹 브라우저 |
| **백그라운드 서비스** | 데몬(daemon): 네트워크 리스너, 프로세스 스케줄러, 에러 모니터, 프린트 서버 |

> 대부분의 사용자는 시스템 콜이 아닌 애플리케이션/시스템 프로그램을 통해 OS를 인식한다.

---

## 2.5 링커와 로더 (Linkers and Loaders)

소스코드에서 메모리 실행까지의 파이프라인:

```
main.c ──[컴파일러]──→ main.o (재배치 가능 오브젝트 파일)
                           │
other.o, libm ─────────────┤
                           │
                      [링커(Linker)]
                           │
                           ▼
                      main (바이너리 실행 파일)
                           │
                      [로더(Loader)]
                           │
                           ▼
                   프로세스 주소 공간 (메모리)
                           │
                    CPU에서 실행 가능
```

- **재배치 가능 오브젝트 파일(relocatable object file)**: 임의의 물리 메모리 주소에 로드될 수 있도록 설계
- **링커(linker)**: 여러 오브젝트 파일을 단일 바이너리 실행 파일로 결합. `-lm` 등 라이브러리 포함
- **로더(loader)**: 바이너리를 메모리에 로드. UNIX에서 `./main` 실행 시: `fork()` → `exec(loader, "main")` 순서로 동작
- **재배치(relocation)**: 프로그램 파트에 최종 주소를 할당하고 해당 주소에 맞게 코드/데이터 조정

### 동적 링킹 (Dynamic Linking)

실행 파일에 라이브러리를 미리 포함하지 않고, **실행 시점에 필요할 때만 로드**하는 방식.

- Windows: **DLL (Dynamically Linked Libraries)**
- 장점: 미사용 라이브러리 로드 방지. 여러 프로세스가 동일 DLL 공유 → 메모리 절약
- 링커가 재배치 정보(relocation info) 삽입 → 실행 시 필요 라이브러리 조건부 링크

### 실행 파일 포맷

| OS | 포맷 |
|----|------|
| Linux/UNIX | **ELF** (Executable and Linkable Format) |
| Windows | **PE** (Portable Executable) |
| macOS | **Mach-O** |

ELF 파일에는 프로그램 진입점(entry point) 주소가 포함됨. `readelf` 명령으로 분석 가능.

---

## 2.6 애플리케이션이 OS 종속적인 이유

동일 애플리케이션을 다른 OS에서 실행하기 어려운 이유:

1. **바이너리 포맷**: 실행 파일의 헤더, 명령어, 변수의 레이아웃이 OS마다 다름 (ELF vs PE vs Mach-O)
2. **CPU 명령어 집합**: x86과 ARMv8은 서로 다른 명령어 집합
3. **시스템 콜**: 번호, 파라미터 방식, 의미, 반환값이 OS마다 상이

### 크로스 플랫폼 접근법

| 방법 | 예시 | 단점 |
|------|------|------|
| **인터프리티드 언어** | Python, Ruby | 성능 저하, OS 기능 일부만 지원 |
| **가상 머신(VM)** | Java (JVM), ART | JVM이 각 OS에 포팅됨. 성능 오버헤드 |
| **표준 API + 각 플랫폼별 컴파일** | POSIX API | 포팅 작업 필요, 버전마다 반복 |

### ABI (Application Binary Interface)

API가 애플리케이션 수준의 인터페이스라면, **ABI는 아키텍처 수준의 바이너리 인터페이스**다.

- 주소 폭, 시스템 콜 파라미터 전달 방식, 런타임 스택 구조, 데이터 타입 크기 등을 명시
- 특정 아키텍처(예: ARMv8) 위에서 정의됨 → ABI가 같으면 다른 시스템에서 실행 가능
- 단, ABI는 플랫폼 간 호환성보다는 **아키텍처 내** 호환성 보장에 초점

---

## 2.7 OS 설계와 구현

### 2.7.1 설계 목표 (Design Goals)

- **사용자 목표**: 편리하고, 배우기 쉽고, 신뢰성 있고, 안전하고, 빠를 것
- **시스템 목표**: 설계·구현·유지보수가 쉽고, 유연하고, 신뢰성 있고, 오류 없고, 효율적일 것

실시간 임베디드 OS(VxWorks)와 대규모 멀티유저 OS(Windows Server)는 요구사항이 근본적으로 다르다. 단일 정답은 없으며 트레이드오프의 연속이다.

### 2.7.2 메커니즘과 정책의 분리 (Mechanism vs Policy)

> **메커니즘(Mechanism)**: "어떻게 할 것인가" — `how`  
> **정책(Policy)**: "무엇을 할 것인가" — `what`

예: 타이머(mechanism)는 CPU 보호를 구현하지만, 타이머를 얼마나 길게 설정할지(policy)는 별개의 결정이다.

**왜 분리가 중요한가?**
- 정책은 장소와 시간에 따라 변한다
- 정책 변경 시 메커니즘을 수정하지 않아도 됨 → 파라미터 재정의만으로 충분
- 예: I/O 집중 프로그램과 CPU 집중 프로그램의 우선순위 정책을 바꿔도 스케줄러 메커니즘은 동일

| OS | 설계 철학 |
|----|----------|
| **마이크로커널** | 정책 없는 기본 메커니즘만 커널에. 정책은 사용자 모듈이나 프로그램이 추가 |
| **Windows/macOS** | 메커니즘과 정책을 커널/라이브러리에 긴밀히 인코딩 → 일관된 UI/UX |
| **Linux** | 특정 스케줄링 메커니즘이 있지만, 오픈소스라 누구나 정책 변경 가능 |

### 2.7.3 구현 (Implementation)

- 초기 OS: 어셈블리어로 작성
- 현재: 대부분 **C/C++** 로 작성. 극히 일부(인터럽트 핸들러, 저수준 루틴)만 어셈블리
- Android: 커널은 C + 어셈블리 / 시스템 라이브러리는 C/C++ / 앱 프레임워크는 Java

**고수준 언어 사용의 장점**: 빠른 개발, 컴팩트한 코드, 디버깅 용이, 컴파일러 최적화 혜택, **이식성(portability)** 향상.

> 성능 개선의 핵심은 어셈블리 최적화보다 **더 나은 자료구조와 알고리즘**이다. 성능 병목은 인터럽트 핸들러, I/O 매니저, 메모리 매니저, CPU 스케줄러에 집중된다.

---

## 2.8 운영체제 구조

### 2.8.1 모놀리식 구조 (Monolithic Structure)

단일 주소 공간에서 동작하는 단일 정적 바이너리. 구조가 없다는 것 자체가 구조다.

**전통적 UNIX 구조**:
```
─────────────────────────────────── 사용자
  shells / compilers / libraries
─────────────────────── system-call interface
  file system │ CPU scheduling │ memory mgmt
  device drivers │ I/O subsystem
─────────────────────── hardware interface
          Hardware
```

**Linux 구조**:
```
         Applications
              │ glibc
    System-Call Interface
  ┌────────────────────────┐
  │  File     │  CPU       │  Memory  │  Devices  │
  │  Systems  │  Scheduler │  Manager │  (drivers)│
  └────────────────────────┘
           Hardware
```

- Linux는 모놀리식이지만 **LKM(Loadable Kernel Module)**을 지원하여 런타임에 기능 추가 가능

**장단점**:
- ✅ 시스템 콜 오버헤드 최소화. 커널 내부 통신이 빠름 → **성능 최우선**
- ❌ 구현과 확장이 어려움. 한 부분의 변경이 다른 부분에 광범위하게 영향 (tightly coupled)

### 2.8.2 계층적 접근 (Layered Approach)

OS를 여러 계층으로 분리. 계층 0 = 하드웨어, 계층 N = 사용자 인터페이스.

```
Layer N  ┌────────────────┐  User Interface
         │   Layer N-1   │
         │   Layer ...   │  ← 각 계층은 하위 계층의 서비스만 사용
         │   Layer 1     │
Layer 0  └────────────────┘  Hardware
```

- **장점**: 구성과 디버깅이 단순함. 계층 M의 오류는 반드시 계층 M에 존재 (하위 계층은 이미 검증)
- **단점**:
  - 각 계층의 기능을 적절히 정의하기 어려움
  - 서비스 요청이 여러 계층을 통과해야 하므로 **성능 저하**
  - 순수 계층 방식을 채택한 OS는 거의 없음. 네트워크(TCP/IP), 웹 앱에서는 성공적으로 사용

### 2.8.3 마이크로커널 (Microkernels)

커널에서 필수적이지 않은 컴포넌트를 제거하고, **사용자 레벨 프로그램으로 구현**.

```
User Space:
┌──────────┐  ┌──────────┐  ┌──────────┐
│   File   │  │  Device  │  │  User    │
│  Server  │  │  Driver  │  │  App     │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │              │              │
     └──────────────┼──────────────┘
                    │  message passing
─────────────── Microkernel ───────────────
     (IPC, 최소 프로세스/메모리 관리만)
─────────────── Hardware ──────────────────
```

**통신**: 서비스들은 마이크로커널을 통한 **메시지 패싱**으로만 상호작용. 클라이언트 프로그램은 파일 서버와 직접 통신하지 않음.

**장점**:
- 확장 용이: 새 서비스는 사용자 공간에 추가 → 커널 수정 불필요
- 이식성(portability) 향상
- 신뢰성/보안: 대부분 서비스가 사용자 프로세스 → 서비스 실패가 OS 전체에 영향 없음

**단점**:
- 서비스 간 통신 시 메시지 복사 + 프로세스 전환 오버헤드 → **성능 저하**
- Windows NT: 마이크로커널로 시작 → 성능 문제로 Windows XP에서 모놀리식에 가까워짐

**대표 사례**:
- **Darwin** (macOS/iOS의 커널): Mach 마이크로커널 + BSD UNIX 커널 복합
- **QNX**: 실시간 임베디드 OS. 메시지 패싱과 프로세스 스케줄링만 마이크로커널이 담당

### 2.8.4 모듈 방식 (Loadable Kernel Modules, LKM)

현대 OS의 가장 보편적인 설계 방법론. 커널이 핵심 컴포넌트를 가지고, **부팅 시 또는 런타임에 모듈을 동적으로 연결**.

```
┌─────────────────────────────────────┐
│           Core Kernel               │
│  (CPU scheduling, memory mgmt)      │
├────────────┬────────────────────────┤
│  Module A  │  Module B  │  Module C │  ← 런타임에 삽입/제거 가능
│  (ext4 FS) │  (USB drv) │  (NIC drv)│
└────────────┴────────────┴───────────┘
```

- 계층 시스템과 유사: 각 커널 섹션에 명확하게 정의된 보호 인터페이스
- 마이크로커널보다 효율적: 모듈 간 통신에 메시지 패싱 불필요 (같은 주소 공간)
- Linux에서 LKM은 디바이스 드라이버와 파일 시스템 지원에 주로 활용
  - USB 장치 연결 시 해당 드라이버 모듈 자동 로드
  - `insmod` / `rmmod` / `lsmod` 명령으로 관리

### 2.8.5 하이브리드 시스템 (Hybrid Systems)

실제 OS는 대부분 여러 구조를 조합한 하이브리드. 성능, 보안, 사용성을 균형 있게 확보.

#### macOS와 iOS (Darwin)

```
┌───────────────────────────────┐
│      User Experience          │  (Aqua / Springboard)
├───────────────────────────────┤
│   Application Frameworks      │  (Cocoa / Cocoa Touch)
├───────────────────────────────┤
│      Core Frameworks          │  (QuickTime, OpenGL)
├───────────────────────────────┤
│   Kernel Environment (Darwin) │
│  ┌─────────────────────────┐  │
│  │   Mach microkernel      │  │  ← IPC, 메모리 관리, CPU 스케줄링
│  ├─────────────────────────┤  │
│  │   BSD UNIX kernel       │  │  ← POSIX 시스템 콜, 파일 시스템
│  ├─────────────────────────┤  │
│  │   I/O Kit + kexts(LKM) │  │  ← 디바이스 드라이버, 커널 확장
│  └─────────────────────────┘  │
└───────────────────────────────┘
         Hardware
```

Darwin은 두 가지 시스템 콜 인터페이스를 제공한다:
- **Mach traps**: 메모리 관리, 스케줄링, IPC (task, thread, port, memory object 추상화)
- **BSD system calls**: POSIX 기능 (`fork()` 등)

**성능 문제 해결**: Mach, BSD, I/O kit, kext를 **단일 주소 공간**에 통합 → 메시지 패싱 시 복사 불필요. 순수 마이크로커널이 아니지만 메시지 패싱 오버헤드를 제거.

macOS vs iOS 차이점:
| | macOS | iOS |
|--|-------|-----|
| 대상 | 데스크톱/랩톱 | 스마트폰/태블릿 |
| CPU | Intel x86 | ARM |
| API 개방 | POSIX/BSD API 공개 | POSIX/BSD API 제한 |
| 보안 설정 | 상대적으로 느슨 | 더 엄격 |

#### Android

```
┌────────────────────────────────────┐
│          Applications              │  Java로 개발
├────────────────────────────────────┤
│       ART (Android RunTime)        │  AOT 컴파일 JVM (.dex 파일)
├────────────────────────────────────┤
│      Android Frameworks            │  Java API
├────────────────────────┬───────────┤
│  Native Libraries       │   JNI    │  webkit, SQLite, OpenGL, SSL
│  (C/C++)               │          │
├────────────────────────┴───────────┤
│   HAL (Hardware Abstraction Layer) │  카메라, GPS, 센서 추상화
├────────────────────────────────────┤
│    Linux Kernel (Modified)         │  Binder IPC, 전력 관리 추가
└────────────────────────────────────┘
         Hardware
```

**핵심 특징**:
- **ART(Android RunTime)**: **AOT(Ahead-of-Time) 컴파일** 적용. `.class` → `.dex` → 설치 시 네이티브 기계어 변환. JIT 대비 실행 효율 향상 + 전력 소비 감소
- **JNI(Java Native Interface)**: 자바에서 네이티브 코드(C/C++) 직접 호출. 이식성 포기 대신 특정 하드웨어 기능 활용
- **HAL**: 하드웨어 독립성 확보 → 다양한 제조사 디바이스 지원
- **Bionic**: glibc 대신 Android 전용 C 라이브러리. 더 작은 메모리 풋프린트, 느린 모바일 CPU 최적화, GPL 라이선스 우회
- **Binder IPC**: Android만의 고성능 IPC 메커니즘 (3.8.2.1절)

---

## 2.9 OS 빌드와 부팅

### 2.9.1 운영체제 생성 (OS Generation)

OS를 처음부터 빌드하는 단계:
1. OS 소스코드 작성 또는 획득
2. 대상 시스템에 맞게 OS 설정 (configuration file 생성)
3. OS 컴파일
4. OS 설치
5. 부팅 및 새 OS 실행

**Linux 빌드 과정**:
```bash
# 1. 소스 다운로드
wget https://www.kernel.org/...

# 2. 커널 설정 (menuconfig UI)
make menuconfig   # → .config 파일 생성

# 3. 커널 컴파일
make              # → vmlinuz (커널 이미지) 생성

# 4. 커널 모듈 컴파일
make modules

# 5. 모듈 설치
make modules_install

# 6. 커널 설치
make install
```

### 2.9.2 시스템 부팅 (System Boot)

커널을 메모리에 로드하여 실행하는 과정:

```
전원 인가
    │
    ▼
┌─────────────┐
│ BIOS / UEFI │  ← 펌웨어. 비휘발성 메모리에 저장
│  (firmware) │    하드웨어 진단(POST) + boot loader 실행
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ Boot Loader │  ← GRUB (Linux/UNIX), LK (Android)
│   (GRUB)   │    커널 이미지 로케이트 및 메모리 로드
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   Kernel    │  ← 압축 해제 후 실행
│ (vmlinuz)   │    initramfs → 실제 root fs 마운트
└──────┬──────┘
       │
       ▼
┌─────────────┐
│   systemd   │  ← Linux 최초 프로세스 (PID 1)
│  (init)     │    서비스 데몬들 시작
└──────┬──────┘
       │
       ▼
   Login Prompt
```

**BIOS vs UEFI**:
| | BIOS | UEFI |
|-|------|------|
| 부트 단계 | 다단계 (멀티스테이지) | 단일 완전한 부트 관리자 |
| 64비트 지원 | 제한적 | 완전 지원 |
| 대용량 디스크 | 제한 (2TB) | 지원 |
| 속도 | 느림 | 빠름 |

**GRUB**: Linux/UNIX의 오픈소스 부트로더. `/proc/cmdline`에서 커널 파라미터 확인 가능:
```
BOOT_IMAGE=/boot/vmlinuz-4.4.0-59-generic
root=UUID=5f2e2232-4e47-4fe8-ae94-45ea749a5c92
```

**Linux 부팅 특이사항**:
- 커널 이미지(`vmlinuz`)는 압축된 파일 → 메모리 로드 후 압축 해제
- **initramfs**: 임시 RAM 파일 시스템. 실제 root fs 마운트에 필요한 드라이버/모듈 포함. 이후 실제 root fs로 교체
- Android: GRUB 미사용. 제조사가 boot loader 제공(주로 LK). initramfs를 root fs로 유지

---

## 2.10 운영체제 디버깅

### 2.10.1 장애 분석 (Failure Analysis)

- **프로세스 실패**: 에러 정보를 **로그 파일**에 기록. **코어 덤프(core dump)** — 프로세스 메모리 스냅샷을 파일로 저장 → 디버거로 분석
- **커널 실패(crash)**: 에러 정보 로그 + **크래시 덤프(crash dump)**
  - 파일 시스템이 손상될 수 있으므로 덤프는 파일 시스템 외부 디스크 영역에 저장
  - 재부팅 후 별도 프로세스가 해당 영역에서 크래시 덤프 파일 생성

### 2.10.2 성능 모니터링과 튜닝

두 가지 접근 방식:

#### 카운터(Counters) — 통계값 조회

| 범위 | Linux 도구 | 기능 |
|------|-----------|------|
| 프로세스별 | `ps` | 프로세스 정보 |
| 프로세스별 | `top` | 실시간 프로세스 통계 |
| 시스템 전체 | `vmstat` | 메모리 사용 통계 |
| 시스템 전체 | `netstat` | 네트워크 인터페이스 통계 |
| 시스템 전체 | `iostat` | 디스크 I/O 사용량 |

Linux 카운터 도구들은 `/proc` 파일 시스템을 읽는다. `/proc`는 커널 메모리에만 존재하는 가상 파일 시스템으로, 프로세스별(`/proc/2155/`) 및 커널 통계를 제공한다.

#### 트레이싱(Tracing) — 이벤트 추적

| 범위 | Linux 도구 | 기능 |
|------|-----------|------|
| 프로세스별 | `strace` | 프로세스의 시스템 콜 추적 |
| 프로세스별 | `gdb` | 소스 레벨 디버거 |
| 시스템 전체 | `perf` | Linux 성능 도구 모음 |
| 시스템 전체 | `tcpdump` | 네트워크 패킷 수집 |

### 2.10.3 BCC (BPF Compiler Collection)

사용자 레벨과 커널 레벨 코드의 상호작용을 안전하게 추적하는 동적 커널 트레이싱 도구.

```
BCC Tool (Python)
      │ 내장 C 코드
      ▼
┌────────────────┐
│  eBPF 명령어   │ ← C의 서브셋으로 작성 → eBPF 명령어로 컴파일
│  (검증 통과 후) │ ← Verifier가 안전성 검증 (성능/보안 영향 확인)
└───────┬────────┘
        │ probes 또는 tracepoints로 삽입
        ▼
┌────────────────┐
│  Linux Kernel  │ ← 동적으로 삽입. 시스템 신뢰성 무영향
└────────────────┘
```

**eBPF**: extended Berkeley Packet Filter. 원래 1990년대 네트워크 트래픽 필터링용으로 개발. 이후 다양한 커널 트레이싱 기능으로 확장.

**BCC 특징**:
- Python 프론트엔드로 eBPF 프로그램 작성 용이
- **프로덕션 시스템에서 live 사용 가능** — 시스템 중단 없이 성능 모니터링
- 시스템 관리자가 병목이나 보안 취약점 실시간 탐지에 활용

**disksnoop 예시**:
```bash
./disksnoop.py
# 출력:
# TIME(s)       T    BYTES    LAT(ms)
# 1946.291867   R    8        0.27
# 1948.345850   W    8192     0.96
```

**특정 프로세스 추적**:
```bash
./opensnoop -p 1225  # PID 1225의 open() 시스템 콜만 추적
```

---

## 핵심 정리

| 개념 | 설명 |
|------|------|
| **시스템 콜** | OS 서비스의 프로그래밍 인터페이스. API를 통해 간접 접근 |
| **RTE** | 시스템 콜 인터페이스를 유지하는 실행 환경 전체 |
| **링커** | 오브젝트 파일들을 단일 실행 파일로 결합 |
| **로더** | 실행 파일을 메모리에 로드. UNIX: fork() → exec() |
| **DLL/동적 링킹** | 런타임에 필요 시 라이브러리 로드. 메모리 절약 |
| **ABI** | 아키텍처 수준의 바이너리 인터페이스. API의 저수준 버전 |
| **메커니즘 vs 정책** | how vs what. 분리가 유연성의 핵심 |
| **모놀리식** | 단일 주소 공간. 성능 최우선. 변경 어려움 |
| **계층적** | 디버깅 단순화. 성능 저하로 실용성 낮음 |
| **마이크로커널** | 최소 커널 + 사용자 공간 서비스. 안전하지만 IPC 오버헤드 |
| **LKM** | 런타임 모듈 삽입/제거. Linux의 핵심 동적 구조 |
| **Darwin** | Mach + BSD를 단일 주소 공간으로 통합한 하이브리드 |
| **AOT 컴파일** | Android ART의 방식. 설치 시 네이티브 코드로 변환 → JIT 대비 효율↑ |
| **GRUB** | Linux/UNIX의 오픈소스 부트로더 |
| **BCC/eBPF** | 동적 커널 트레이싱. 프로덕션 환경에서 안전하게 사용 가능 |
