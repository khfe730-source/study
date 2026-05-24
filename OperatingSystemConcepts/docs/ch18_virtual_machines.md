# Chapter 18: Virtual Machines

## 개요

가상 머신(Virtual Machine)은 단일 물리 하드웨어를 복수의 실행 환경으로 추상화하는 기술이다. 각 게스트(Guest)는 자신이 전용 하드웨어 위에서 실행된다고 믿지만, 실제로는 VMM이 자원을 공유·보호·관리한다.

```
┌─────────────────────────────────────┐
│  processes  processes  processes    │
│  ┌────────┐ ┌────────┐ ┌────────┐  │
│  │ kernel │ │ kernel │ │ kernel │  │
│  │  VM1   │ │  VM2   │ │  VM3   │  │
│  └────────┘ └────────┘ └────────┘  │
│      Virtual Machine Manager (VMM) │
│──────────────────────────────────  │
│           hardware                  │
└─────────────────────────────────────┘
         (b) 가상 머신 모델

┌───────────────┐
│   processes   │
│  ┌─────────┐  │
│  │  kernel │  │
│  └─────────┘  │
│   hardware    │
└───────────────┘
    (a) 비가상 모델
```

**핵심 용어**

| 용어 | 설명 |
|------|------|
| Host | VMM이 실행되는 실제 하드웨어 시스템 |
| VMM / Hypervisor | 가상 머신을 생성·실행·관리하는 소프트웨어 |
| Guest | VMM 위에서 실행되는 OS 또는 애플리케이션 |
| VCPU | VMM이 각 게스트를 위해 유지하는 가상 CPU 상태 구조 |

---

## 18.1 역사 (History)

- **1972**: IBM VM/370 — 최초 상업적 가상 머신. 메인프레임에 복수 OS 동시 실행
  - **Minidisk**: 물리 디스크 트랙의 일부를 가상 디스크로 제공
  - 사용자는 CMS(단일 사용자 인터랙티브 OS) 실행

- **Popek-Goldberg 요구사항 (1974)**: VMM 구현의 형식적 정의
  - **Fidelity**: VMM 환경은 원래 머신과 본질적으로 동일해야 함
  - **Performance**: 성능 저하가 미미해야 함
  - **Safety**: VMM이 시스템 자원을 완전히 통제해야 함

- **1990년대 후반**: Intel 80x86 대중화 → Xen, VMware가 x86 가상화 기술 개발

---

## 18.2 이점과 특징 (Benefits and Features)

- **격리(Isolation)**: 각 VM은 독립적 → 보안·안정성 향상 (바이러스 연구, 테스트 환경)
- **스냅샷/클론(Snapshot/Clone)**: 실행 중 상태를 저장·복원·복제 가능
- **OS 연구·개발**: 가상 머신 내에서 커널 변경 테스트 → 프로덕션 시스템 영향 없음
- **시스템 통합(Consolidation)**: 여러 저활용 서버를 하나의 물리 서버에 통합 → 자원 최적화
- **템플릿(Templating)**: 표준 VM 이미지를 복제하여 신속 배포
- **라이브 마이그레이션(Live Migration)**: 실행 중 VM을 다른 물리 서버로 이동 → 무중단 서비스
- **클라우드 컴퓨팅**: VM을 API로 수천 개 생성·관리 (AWS, GCP 등)
- **데스크탑 가상화**: 원격 데이터센터의 VM에 RDP로 접속 → 로컬에 데이터 없음

---

## 18.3 핵심 구성 요소 (Building Blocks)

### 18.3.1 Trap-and-Emulate

```
┌─────────────────────────────────────────────────┐
│  guest user mode                                │
│  ┌──────────────────────────┐                  │
│  │  user processes          │                  │
│  └──────────────────────────┘                  │
│  ┌──────────────────────────┐                  │
│  │  operating system        │  ← 특권 명령 실행 시도│
│  └──────────┬───────────────┘                  │
│             │ trap (user mode에서 특권 명령 → 예외) │
│  ┌──────────▼───────────────┐                  │
│  │  VMM (kernel mode)       │                  │
│  │  emulate action          │  VCPU 상태 업데이트│
│  │  return to guest         │                  │
│  └──────────────────────────┘                  │
└─────────────────────────────────────────────────┘
```

- **원리**: 게스트 OS가 특권 명령(Privileged Instruction) 실행 시도 → trap → VMM이 에뮬레이션
- **문제점**: x86의 특정 명령어(e.g., `popf`)는 user mode에서 실행해도 trap이 발생하지 않음 → "Special Instructions" 문제

### 18.3.2 Binary Translation (이진 변환)

```
┌─────────────────────────────────────────────────┐
│  guest VCPU in kernel mode                      │
│  ┌──────────────────────────┐                  │
│  │  VMM reads instructions  │ ← program counter│
│  │  from guest              │                  │
│  └──────────┬───────────────┘                  │
│             │                                   │
│  일반 명령어 ──→ native 실행                      │
│  특수 명령어 ──→ translate → 동등한 명령 시퀀스     │
│             │                VCPU 상태 변경 포함  │
│  ┌──────────▼───────────────┐                  │
│  │  Translation Cache       │ ← 캐시 재활용      │
│  └──────────────────────────┘                  │
└─────────────────────────────────────────────────┘
```

- **동작**: VCPU가 kernel mode일 때 VMM이 명령어를 읽어 특수 명령어만 변환
- **성능 최적화**: Translation Cache — 한 번 변환된 명령은 캐시에 저장하여 재사용
- **VMware 실증**: Windows XP 부팅/종료 시 950,000회 변환, 변환당 3μs → 총 오버헤드 약 5%

### 18.3.3 Nested Page Tables (NPT)

VMM은 게스트의 페이지 테이블 상태를 NPT로 유지한다:

```
Guest Virtual Address
      │
      ▼
Guest Page Table (guest가 관리한다고 믿는)
      │
      ▼
Nested Page Table (VMM이 관리)
      │
      ▼
Host Physical Address
```

- 게스트 page-table 변경 → VMM이 가로채어 NPT도 갱신
- TLB miss 증가 가능 → 성능 손실 단점

### 18.3.4 하드웨어 지원 (Hardware Assistance)

| 기술 | 제조사 | 설명 |
|------|--------|------|
| VT-x | Intel (2005~) | root/nonroot 모드, VMCS(VM Control Structure), binary translation 불필요 |
| AMD-V | AMD (2006~) | host/guest 모드, VCPU 상태 자동 저장/복원 |
| EPT (Extended Page Tables) | Intel | 하드웨어 NPT — guest virtual → host physical 변환 가속 |
| RVI (Rapid Virtualization Indexing) | AMD | AMD 버전의 하드웨어 NPT |
| VT-d | Intel | DMA 가상화 — protection domain으로 게스트별 메모리 접근 제한, DMA 주소 변환 |
| Interrupt Remapping | Intel | 인터럽트를 해당 게스트가 실행 중인 코어로 자동 라우팅 |
| EL2 + HVC | ARM v8 | 커널(EL1)보다 더 높은 특권 레벨로 하이퍼바이저 격리. HVC 명령으로 게스트 → 하이퍼바이저 호출 |
| Thin Hypervisor | macOS | `HyperVisor.framework` — 커널이 특권 가상화 명령 대신 실행, 커널 모듈 불필요 |

---

## 18.4 가상 머신 유형 (Types of VMs)

### Type 0 Hypervisor (펌웨어)

```
┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
│guest 1 │ │guest 2 │ │guest 3 │ │guest 4 │
│ CPUs   │ │ CPUs   │ │ CPUs   │ │ CPUs   │
│ memory │ │ memory │ │ memory │ │ memory │
└────────┘ └────────┘ └────────┘ └────────┘
        hypervisor (in firmware)      I/O
              ────────────────────────────
                    hardware
```

- **펌웨어** 구현, 부팅 시 로드
- 각 파티션에 전용 CPU·메모리·I/O 할당 → 게스트는 전용 하드웨어 보유 착각
- 예: IBM LPARs, Oracle LDOMs
- 특징: 기능 셋이 가장 작음. Type 0 하이퍼바이저 위에서도 가상화 가능

### Type 1 Hypervisor (운영체제 수준)

- **네이티브 하드웨어에서 실행**되는 특수 목적 OS
- 게스트 OS를 프로세스처럼 관리하되 특수 처리
- CPU 스케줄링·메모리 관리·I/O 관리 직접 수행
- 예: VMware ESX(상업), Citrix XenServer, Windows Hyper-V, Red Hat KVM
- **데이터센터의 OS** 역할 — 수백 개의 VM 통합 관리, 라이브 마이그레이션 지원

### Type 2 Hypervisor (애플리케이션 수준)

- **호스트 OS 위의 일반 프로세스**로 실행
- 호스트 OS는 가상화가 일어나는 것을 모름
- 하드웨어 지원 기능(관리자 권한 필요) 사용 제한 → 성능 상대적으로 낮음
- 예: VMware Workstation/Fusion, Parallels Desktop, Oracle VirtualBox
- 장점: 다양한 호스트 OS 지원, 호스트 OS 변경 불필요, 학습·테스트 목적에 적합

```
┌──────────┐ ┌──────────┐ ┌──────────┐
│application│ │application│ │application│  ← guest OS 위 앱
│guest OS  │ │guest OS  │ │guest OS  │
│virtualCPU│ │virtualCPU│ │virtualCPU│
│virtualMEM│ │virtualMEM│ │virtualMEM│
│virtualDEV│ │virtualDEV│ │virtualDEV│
└──────────┘ └──────────┘ └──────────┘
         virtualization layer
         host OS (Linux, Windows 등)
    CPU   Memory   I/O devices
```
*(VMware Workstation 구조)*

### Paravirtualization (반가상화)

- 게스트 OS를 **수정**하여 VMM과 협력 → 더 효율적인 실행
- **Xen** 방식:
  - **I/O**: 게스트-VMM 간 공유 메모리 내 순환 버퍼(circular buffer) → DMA 없이 직접 데이터 교환
  - **메모리**: 게스트 페이지 테이블을 read-only로 설정 → 변경 시 **Hypercall**(게스트→VMM 호출)로 요청
  - x86 binary translation 없이 가상화 (대신 게스트 OS 수정 필요)
  - 현재는 하드웨어 지원으로 게스트 OS 수정 불필요

### Programming-Environment Virtualization

- 프로그래밍 언어 자체가 가상 실행 환경을 정의
- **JVM (Java Virtual Machine)**: Java 프로그램 → 아키텍처 독립적 바이트코드(.class) → JVM이 어디서든 실행
- **Microsoft .NET**: CLR(Common Language Runtime)

### Emulation (에뮬레이션)

- 게스트의 ISA(명령어 집합)가 호스트와 **다른 경우** 사용
- 모든 명령어를 소프트웨어로 해석·변환 → 성능 10배 이상 저하
- 용도: 구형 게임 콘솔 에뮬레이터, 레거시 시스템 유지

### Application Containment (애플리케이션 컨테이너)

- 완전 가상화 없이 OS 레벨의 격리만 제공
- 단일 커널 공유, 하드웨어 가상화 없음 → 경량화

| 기술 | OS | 설명 |
|------|-----|------|
| Zones/Containers | Oracle Solaris 10 | 가상 플랫폼 레이어, 독립 네트워크·사용자 계정 |
| Jails | FreeBSD | 최초의 컨테이너 유사 기능 |
| WPARs | IBM AIX | 유사 기능 |
| LXC | Linux | clone() 플래그로 구현 (2014~) |
| Docker | Linux | LXC 기반, 이미지·레지스트리 생태계 |
| Kubernetes | 다중 OS | Docker 오케스트레이션 — 자동 배포·스케일링·관리 |

---

## 18.5 가상화와 OS 구성 요소

### 18.5.1 CPU 스케줄링

- VMM은 물리 CPU를 가상 CPU(VCPU)로 다중화
- **Over-commitment**: 게스트에 설정된 VCPU 수 > 물리 CPU 수 → VMM이 비례 배분
  ```
  예) 물리 CPU 6개, 게스트 VCPU 합계 12개
      → 각 게스트 CPU 자원의 50% 실제 할당
  ```
- **문제**: 게스트 OS의 100ms 타임슬라이스가 실제로는 수 초 걸릴 수 있음 → time-of-day 클록 오차
- **해결**: 게스트 OS용 보조 애플리케이션(clock drift 교정)

### 18.5.2 메모리 관리

VMM은 메모리를 **과도 할당(overcommit)** — 총 게스트 설정 메모리 > 물리 메모리

**VMware ESX의 3가지 메모리 회수 기법**:

1. **Double Paging**: VMM이 자체 page-replacement 알고리즘 실행 → 게스트가 physical memory라고 믿는 공간이 실제로는 VMM의 backing store일 수 있음

2. **Balloon Process (풍선 프로세스)**:
   ```
   VMM ──(메모리 필요 신호)──▶ Balloon Driver (게스트 내)
   Balloon Driver가 게스트 메모리 고정(pin) → 게스트가 메모리 부족으로 자체 page-out
   VMM이 고정된 페이지를 실제 물리 메모리로 재사용
   메모리 압박 해소 시 → Balloon Driver가 해제(unpins)
   ```

3. **페이지 공유(Memory Deduplication)**:
   - VMM이 게스트 메모리를 무작위 샘플링 → 각 페이지의 해시("thumbprint") 생성
   - 해시 일치 → 바이트 단위 비교 → 동일 시 하나만 유지, 나머지는 물리 매핑 공유
   - 동일 OS를 실행하는 복수 게스트 → OS 페이지 하나만 물리 메모리에 유지

### 18.5.3 I/O

| I/O 방식 | 설명 |
|---------|------|
| 전용 I/O (Direct Access) | 게스트에게 I/O 장치 독점 할당. Type 0에서 native 수준 성능 |
| 공유 I/O | VMM이 모든 I/O 검사·라우팅 → 보호와 격리 |
| 이상화 장치 드라이버 | 게스트는 단순 가상 장치 사용 → VMM이 내부적으로 복잡한 실제 드라이버에 전달 |
| DMA Pass-through (VT-d) | VMM이 protection domain 설정 → 게스트-장치 간 DMA 직접 통과 |
| 인터럽트 리매핑 | 인터럽트를 해당 게스트가 실행 중인 코어로 자동 전달 |

**네트워킹**:
- **Bridging**: 게스트에 독립 IP 할당 → 외부 네트워크 직접 노출
- **NAT**: VMM이 라우팅 → 게스트는 서버 로컬 주소만 가짐
- VMM이 가상 스위치 역할 → 내부 게스트 간, 게스트-외부 간 패킷 라우팅·방화벽

### 18.5.4 스토리지 관리

- 게스트 루트 디스크 = VMM 파일 시스템 내 **디스크 이미지 파일**
- 이미지 복사 = 게스트 복제; 이미지 이동 = 게스트 이전
- **P-to-V (Physical-to-Virtual)**: 물리 시스템 → VM 이미지 변환
- **V-to-P (Virtual-to-Physical)**: VM → 물리 시스템으로 역변환 (디버깅 목적)

### 18.5.5 Live Migration (라이브 마이그레이션)

서비스 중단 없이 실행 중 VM을 다른 물리 서버로 이동:

```
단계별 라이브 마이그레이션

0: source에서 guest 실행 중
1: source VMM ──연결 확립──▶ target VMM
2: target VMM: 새 VCPU + NPT + 상태 저장소 생성
3: source ──R/O 메모리 페이지 전송──▶ target
4: source ──R/W 페이지 전송(clean 표시)──▶ target
5: source ──dirty 페이지 반복 전송──▶ target (dirty 주기 단축)
6: dirty 주기가 매우 짧아지면:
   source가 guest 동결(freeze)
   → 최종 VCPU 상태 + dirty 페이지 전송
   → target이 guest 실행 시작
7: source가 guest 종료
```

- **MAC 주소 이동성**: 네트워크 스위치가 MAC → 물리 위치 매핑 업데이트
- **디스크 제외**: 디스크 I/O는 마이그레이션 범위 밖 → 원격 스토리지(NFS/CIFS/iSCSI) 필수

---

## 18.6 가상화 연구 (Virtualization Research)

### Unikernel

- 애플리케이션 + 필요한 시스템 라이브러리 + 커널 서비스를 **단일 바이너리**로 컴파일
- **단일 주소 공간(single address space)** 내에서 VM 환경에서 실행
- 장점: 공격 표면 축소, 자원 풋프린트 감소, 배포 효율성
- 대표: unikernel.org

### Partitioning Hypervisor (분리 하이퍼바이저)

- 물리 자원을 over-commit 없이 게스트에 **완전 분리 할당**
- 하이퍼바이저는 초기화·시작만 수행 → 이후 운영에 개입 없음
- 각 VM이 자신의 할당된 하드웨어를 간섭 없이 관리 → **실시간 처리 가능**
- 대표: Quest-V, eVM, Xtratum, Siemens Jailhouse
- 활용: 로봇공학, 자율주행차, IoT

---

## 18.7 요약

| 구분 | Type 0 | Type 1 | Type 2 | Paravirt | Emulation | Container |
|------|--------|--------|--------|---------|-----------|-----------|
| 구현 위치 | 펌웨어 | 네이티브 OS | 호스트 앱 | 수정 게스트 | 소프트웨어 | OS 레이어 |
| 예시 | IBM LPAR | VMware ESX, KVM | VirtualBox | Xen | QEMU | Docker |
| 성능 | 최고 | 높음 | 중간 | 높음 | 낮음 | 매우 높음 |
| 커널 공유 | 아니오 | 아니오 | 아니오 | 아니오 | 아니오 | 예 |
| 게스트 OS 수정 | 불필요 | 불필요 | 불필요 | 필요 | 불필요 | 불필요 |

**VMM 구현의 핵심 과제**: 특권 명령어 처리 (trap-and-emulate → binary translation → 하드웨어 지원 순으로 발전)

**현대 가상화의 3대 축**:
1. 하드웨어 지원 (Intel VT-x/AMD-V/EPT/VT-d)
2. 효율적 메모리 관리 (NPT/balloon/dedup)
3. 라이브 마이그레이션 (데이터센터 탄력적 운영)
