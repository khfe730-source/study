# Chapter 7: Linking

## 핵심 주제

링킹(linking)은 여러 오브젝트 파일과 라이브러리를 하나의 실행 가능한 파일로 결합하는 과정이다. 컴파일 타임, 로드 타임, 런타임에 수행될 수 있다. 링킹을 이해해야 하는 이유:
- **큰 프로그램 개발**: 분리 컴파일(separate compilation)로 모듈별 독립 빌드 가능
- **버그 회피**: 중복 전역 심볼 오염, 라이브러리 순서 오류 등 숨은 버그 방지
- **스코프 이해**: `static` 키워드의 링커 수준 동작 이해
- **공유 라이브러리 활용**: 동적 링킹, 런타임 로딩, 인터포지셔닝

---

## 7.1 Compiler Drivers

`gcc`는 컴파일러 드라이버로, 전체 컴파일 파이프라인을 자동 관리한다:

```
gcc -Og -o prog main.c sum.c
```

```
main.c  ──→  [cpp]  ──→  main.i  (전처리)
                ↓
            [cc1]   ──→  main.s  (컴파일)
                ↓
            [as]    ──→  main.o  (어셈블)

sum.c   ──→  [cpp/cc1/as]  ──→  sum.o

main.o + sum.o + [시스템 오브젝트]  ──→  [ld]  ──→  prog (실행 파일)
```

각 단계 명령:
```bash
cpp  main.c /tmp/main.i              # 전처리기
cc1  /tmp/main.i -Og -o /tmp/main.s  # 컴파일러
as   -o /tmp/main.o /tmp/main.s      # 어셈블러
ld   -o prog [시스템 파일] main.o sum.o  # 링커
```

실행 시 셸은 OS의 **로더(loader)** 를 호출하여 `prog`를 메모리에 복사하고 제어를 넘긴다.

---

## 7.2 Static Linking

정적 링커(Linux의 `ld`)는 두 가지 핵심 작업을 수행한다:

**Step 1. 심볼 해석 (Symbol Resolution)**
각 심볼 참조를 정확히 하나의 심볼 정의에 연결한다. 심볼 = 함수, 전역 변수, static 변수.

**Step 2. 재배치 (Relocation)**
컴파일러/어셈블러는 주소 0부터 시작하는 코드와 데이터를 생성한다. 링커가 각 심볼에 실제 런타임 메모리 주소를 할당하고, 모든 심볼 참조를 해당 주소로 수정한다.

> 링커는 오브젝트 파일들을 단순히 바이트 블록의 집합으로 다룬다. 컴파일러/어셈블러가 이미 대부분의 작업을 수행했고, 링커는 최소한의 타깃 머신 이해만으로 동작한다.

---

## 7.3 Object Files

오브젝트 파일의 세 가지 형태:

| 종류 | 설명 | 생성 |
|------|------|------|
| **Relocatable object file** (.o) | 다른 재배치 가능 파일들과 결합하여 실행 파일 생성에 사용 | 컴파일러/어셈블러 |
| **Executable object file** | 메모리에 직접 복사하여 실행 가능 | 링커 |
| **Shared object file** (.so) | 로드 타임 또는 런타임에 동적으로 링크 가능한 특수 재배치 파일 | 컴파일러/링커 |

**오브젝트 파일 포맷:**
- Unix 초기: `a.out` 포맷 (이름이 지금도 남아 있음)
- Windows: PE (Portable Executable)
- macOS: Mach-O
- Linux/Unix x86-64: **ELF (Executable and Linkable Format)**

---

## 7.4 Relocatable Object Files (ELF 구조)

```
ELF 재배치 가능 오브젝트 파일 레이아웃:
┌──────────────────────┐  ← offset 0
│     ELF header       │  (워드 크기, 바이트 순서, 파일 타입, 진입점 등)
├──────────────────────┤
│       .text          │  컴파일된 기계어 코드
├──────────────────────┤
│      .rodata         │  읽기 전용 데이터 (printf 포맷 문자열, switch 점프 테이블)
├──────────────────────┤
│       .data          │  초기화된 전역/정적 C 변수
├──────────────────────┤
│       .bss           │  미초기화 전역/정적 변수 (디스크 공간 0 bytes - placeholder만)
├──────────────────────┤
│      .symtab         │  심볼 테이블 (함수·전역 변수 정의/참조 정보)
├──────────────────────┤
│     .rel.text        │  .text의 재배치 항목 (외부 함수 호출, 전역 변수 참조)
├──────────────────────┤
│     .rel.data        │  전역 변수의 재배치 항목
├──────────────────────┤
│      .debug          │  디버깅 심볼 테이블 (-g 옵션 시)
├──────────────────────┤
│       .line          │  C 소스 라인 번호 ↔ 기계어 매핑 (-g 옵션 시)
├──────────────────────┤
│      .strtab         │  .symtab, .debug의 심볼 이름 문자열 테이블
├──────────────────────┤
│  Section header table│  각 섹션의 위치·크기 기술
└──────────────────────┘
```

**.bss** (Better Save Space): 미초기화 변수는 파일에 실제 데이터가 없고, 런타임에 0으로 초기화된 메모리만 할당된다.

---

## 7.5 Symbols and Symbol Tables

**링커 심볼의 세 종류:**

| 종류 | 정의 | C 대응 |
|------|------|--------|
| **Global symbol (정의)** | 모듈 m이 정의, 다른 모듈이 참조 가능 | non-static 함수, 전역 변수 |
| **Global symbol (참조, external)** | 모듈 m이 참조하지만 다른 모듈에서 정의 | 외부 모듈의 non-static 함수/변수 |
| **Local symbol** | 모듈 m 내부에서만 정의·참조 | `static` 함수, `static` 전역 변수 |

> 중요: 링커의 로컬 심볼 ≠ C 지역 변수. `.symtab`에는 스택 관리되는 지역 변수는 없다.

**static 지역 변수의 심볼 처리:**
```c
int f() { static int x = 0; return x; }  // 컴파일러: x.1 심볼 생성
int g() { static int x = 1; return x; }  // 컴파일러: x.2 심볼 생성
// → .data/.bss에 고유한 이름으로 저장 (스택 아님)
```

**ELF 심볼 테이블 엔트리 구조:**
```c
typedef struct {
    int   name;      // 문자열 테이블 오프셋
    char  type:4,    // FUNC 또는 DATA (4 bits)
          binding:4; // LOCAL 또는 GLOBAL (4 bits)
    char  reserved;
    short section;   // 섹션 헤더 테이블 인덱스
    long  value;     // 섹션 내 오프셋 또는 절대 주소
    long  size;      // 오브젝트 크기 (bytes)
} Elf64_Symbol;
```

**특수 가상 섹션(pseudosection):**
| 섹션 | 의미 |
|------|------|
| `ABS` | 재배치 불필요 심볼 |
| `UNDEF` | 정의되지 않은(외부) 심볼 |
| `COMMON` | 아직 할당되지 않은 미초기화 전역 데이터 |

**COMMON vs .bss (gcc 규칙):**
```
COMMON: 미초기화 전역 변수     (int x;)
.bss:   미초기화 static 변수  (static int x;)
        0으로 초기화된 전역/static 변수 (int x = 0;)
```

**readelf로 심볼 테이블 확인:**
```bash
$ readelf -s main.o
# Num: Value   Size  Type   Bind  Ndx  Name
#   8: 0x0     24    FUNC   GLOBAL  1  main   (.text)
#   9: 0x0      8    OBJECT GLOBAL  3  array  (.data)
#  10: 0x0      0    NOTYPE GLOBAL UND  sum   (미정의 외부)
```

---

## 7.6 Symbol Resolution

### 7.6.1 중복 심볼 이름 해석 규칙

컴파일러는 전역 심볼을 **strong** 또는 **weak**으로 어셈블러에 전달한다:
- **Strong**: 함수, 초기화된 전역 변수
- **Weak**: 미초기화 전역 변수

**Linux 링커 규칙:**
| 규칙 | 상황 | 처리 |
|------|------|------|
| Rule 1 | 동일 이름 strong 심볼 복수 | **오류** |
| Rule 2 | strong 1개 + weak 복수 | **strong 선택** |
| Rule 3 | weak 복수 | **임의 선택** |

**Rule 2/3의 위험성 (조용한 버그):**
```c
// foo5.c
int y = 15212;
int x = 15213;   // ← strong (int, 4 bytes)
int main() { f(); printf("x=0x%x y=0x%x\n", x, y); }

// bar5.c
double x;        // ← weak (double, 8 bytes) → Rule 2로 foo5의 x 선택
void f() { x = -0.0; }
// 결과: double(8bytes) 쓰기가 int x와 int y 모두를 덮어씀!
// 경고: "alignment 4 of symbol 'x' is smaller than 8"
// 출력: x = 0x0  y = 0x80000000  (오염!)
```

**대응:** `gcc -fno-common` (중복 전역 심볼 → 오류) 또는 `-Werror` (경고 → 오류)

### 7.6.2 Static Libraries

정적 라이브러리는 관련 오브젝트 파일들을 **아카이브(.a)** 형식으로 묶은 것이다. 링커는 실제로 참조되는 오브젝트 모듈만 실행 파일에 포함한다.

```bash
# 정적 라이브러리 생성
gcc -c addvec.c multvec.c
ar rcs libvector.a addvec.o multvec.o

# 정적 링킹
gcc -static -o prog2c main2.o ./libvector.a
# 또는
gcc -static -o prog2c main2.o -L. -lvector
```

```
컴파일 → main2.o + libvector.a(addvec.o + multvec.o) + libc.a
링커 → addvec.o와 printf.o만 포함 (multvec.o는 제외)
```

### 7.6.3 정적 라이브러리로 참조 해석 알고리즘

링커는 커맨드라인을 **왼쪽에서 오른쪽**으로 스캔하며 세 집합을 유지:
- **E**: 실행 파일에 합쳐질 재배치 오브젝트 파일 집합
- **U**: 미해석(unresolved) 심볼 집합
- **D**: 이미 정의된 심볼 집합

```
각 입력 파일 f에 대해:
  if f가 오브젝트 파일:
    E에 추가, U와 D 갱신
  if f가 아카이브(.a):
    U의 심볼을 해석할 수 있는 멤버 m을 반복 추가
    → 더 이상 변화 없을 때까지 반복 (고정점)
    → E에 없는 멤버는 버림

스캔 종료 후 U가 비어있지 않으면 오류
```

**라이브러리 순서 오류의 함정:**
```bash
# 오류: 라이브러리가 참조 오브젝트보다 앞에 있으면 U가 비어있어 아무것도 로드 안 됨
gcc -static ./libvector.a main2.o  # → undefined reference to 'addvec'

# 올바른 순서: 오브젝트 파일 먼저, 라이브러리 나중
gcc -static main2.o ./libvector.a

# 상호 의존 시 반복 지정 필요
gcc foo.c libx.a liby.a libx.a   # libx.a ↔ liby.a 순환 의존
```

---

## 7.7 Relocation

심볼 해석 완료 후 링커는 재배치를 수행한다.

### 7.7.1 Relocation Entries

어셈블러는 최종 주소를 모르는 참조를 만날 때 **재배치 엔트리**를 생성한다:

```c
typedef struct {
    long offset;    // 수정이 필요한 참조의 섹션 내 오프셋
    long type:32,   // 재배치 타입
         symbol:32; // 심볼 테이블 인덱스
    long addend;    // 재배치 값 계산 시 더할 상수
} Elf64_Rela;
```

**두 가지 주요 재배치 타입 (x86-64):**

| 타입 | 의미 | 사용처 |
|------|------|--------|
| `R_X86_64_PC32` | 32-bit PC-상대 주소 재배치 | `call` 명령어 (함수 호출) |
| `R_X86_64_32` | 32-bit 절대 주소 재배치 | `mov` 명령어 (전역 변수 주소) |

### 7.7.2 재배치 알고리즘

```c
foreach section s {
    foreach relocation entry r {
        refptr = s + r.offset;   // 수정할 참조의 포인터

        // PC-상대 참조 재배치
        if (r.type == R_X86_64_PC32) {
            refaddr = ADDR(s) + r.offset;  // 참조의 런타임 주소
            *refptr = (unsigned)(ADDR(r.symbol) + r.addend - refaddr);
        }

        // 절대 참조 재배치
        if (r.type == R_X86_64_32)
            *refptr = (unsigned)(ADDR(r.symbol) + r.addend);
    }
}
```

**PC-상대 재배치 예시 (main → sum 호출):**
```
main.o의 call 명령: e8 00 00 00 00  (sum 주소가 0x0으로 placeholder)
재배치 엔트리: r.offset=0xf, r.symbol=sum, r.type=PC32, r.addend=-4

링커가 결정:
  ADDR(.text) = 0x4004d0
  ADDR(sum)   = 0x4004e8

계산:
  refaddr = 0x4004d0 + 0xf = 0x4004df
  *refptr = 0x4004e8 + (-4) - 0x4004df = 0x5

결과: 4004de: e8 05 00 00 00  callq 4004e8 <sum>

CPU 실행 시: PC = 0x4004e3 (call 다음 명령)
           PC ← PC + 0x5 = 0x4004e8 ✓
```

**절대 재배치 예시 (array 주소 로드):**
```
main.o의 mov 명령: bf 00 00 00 00  (array 주소 placeholder)
재배치 엔트리: r.offset=0xa, r.symbol=array, r.type=32, r.addend=0

  ADDR(array) = 0x601018
  *refptr = 0x601018 + 0 = 0x601018

결과: 4004d9: bf 18 10 60 00  mov $0x601018,%edi
```

---

## 7.8 Executable Object Files

```
ELF 실행 파일 레이아웃:
┌──────────────────────┐
│     ELF header       │  진입점(entry point) 주소 포함
├──────────────────────┤
│  Segment header table│  파일 내 연속 청크 → 런타임 메모리 세그먼트 매핑
├──────────────────────┤
│  Read-only segment   │  .init, .text, .rodata (r-x 권한)
│  (코드 세그먼트)     │
├──────────────────────┤
│  Read/write segment  │  .data, .bss (rw- 권한)
│  (데이터 세그먼트)   │
├──────────────────────┤
│      .symtab         │  (메모리에 로드 안 됨)
│      .debug          │
│      .strtab         │
├──────────────────────┤
│  Section header table│
└──────────────────────┘
```

**세그먼트 정렬 요건:**
```
vaddr mod align = off mod align
(align = 2^21 = 0x200000)

예: 데이터 세그먼트
  vaddr = 0x600df8, align = 0x200000
  0x600df8 mod 0x200000 = 0xdf8 = off mod 0x200000  ✓
```

이 정렬은 OS의 가상 메모리 페이지 단위 효율적 전송을 위함이다 (Chapter 9에서 자세히).

---

## 7.9 Loading Executable Object Files

```bash
linux> ./prog
```

셸은 **로더(loader)** 를 호출 (`execve` 시스템 콜):

```
런타임 메모리 이미지 (x86-64 Linux):
┌────────────────────────────────┐ 2^48 - 1
│   Kernel memory (보이지 않음)  │
├────────────────────────────────┤
│      User stack                │ ← %rsp (아래로 성장)
│      (런타임 생성)             │
├────────────────────────────────┤
│   Memory-mapped region         │
│   (공유 라이브러리)            │
├────────────────────────────────┤ ← brk
│   Run-time heap                │
│   (malloc으로 위로 성장)       │
├────────────────────────────────┤
│   .data, .bss (R/W 세그먼트)   │
├────────────────────────────────┤
│   .init/.text/.rodata (R/X)    │
└────────────────────────────────┘ 0x400000
```

**로딩 프로세스 (실제 동작):**
```
fork() → 자식 프로세스 생성
execve() → 로더 호출
  → 기존 가상 메모리 세그먼트 삭제
  → 새 코드/데이터/힙/스택 세그먼트 생성 (페이지 매핑, Copy-on-Demand)
  → _start 함수 진입 (crt1.o에 정의됨)
    → __libc_start_main() 호출 (libc.so)
      → 실행 환경 초기화
      → main() 호출
      → 반환값 처리
```

> ASLR(Address Space Layout Randomization)로 스택/공유 라이브러리/힙 세그먼트의 시작 주소가 실행마다 달라진다. 단, 상대적 위치는 동일하다.

---

## 7.10 Dynamic Linking with Shared Libraries

**정적 라이브러리의 문제:**
1. 업데이트 시 실행 파일 재링크 필요
2. `printf` 같은 공통 코드가 모든 실행 중인 프로세스에 중복 복사됨

**공유 라이브러리(Shared Library, .so):** 로드 타임 또는 런타임에 임의 메모리 주소에 로드하고 프로그램과 동적 링크한다.

```bash
# 공유 라이브러리 생성
gcc -shared -fpic -o libvector.so addvec.c multvec.c
# -fpic: Position-Independent Code 생성
# -shared: 공유 오브젝트 파일 생성

# 공유 라이브러리와 링크
gcc -o prog2l main2.c ./libvector.so
# → prog2l에 libvector.so의 코드/데이터는 포함되지 않음
#   재배치·심볼 테이블 정보만 포함
```

**로드 타임 동적 링킹 과정:**
```
1. 로더가 prog2l 로드
2. .interp 섹션에서 동적 링커 경로 발견 (ld-linux.so)
3. 동적 링커 실행:
   - libc.so 텍스트/데이터를 메모리 세그먼트에 재배치
   - libvector.so 텍스트/데이터를 다른 메모리 세그먼트에 재배치
   - prog2l의 libc.so/libvector.so 참조를 해석
4. 동적 링커가 애플리케이션에 제어 이전
```

**공유 라이브러리의 두 가지 "공유" 의미:**
1. 파일 시스템에 `.so` 파일이 하나만 존재 (정적 라이브러리처럼 실행 파일에 복사 안 됨)
2. 메모리 내 `.text` 섹션 하나를 여러 프로세스가 공유

---

## 7.11 Loading and Linking Shared Libraries from Applications

실행 중인 애플리케이션이 런타임에 공유 라이브러리를 동적으로 로드/링크할 수 있다 (`dlfcn.h`):

```c
#include <dlfcn.h>

// 공유 라이브러리 로드
void *dlopen(const char *filename, int flag);
// flag: RTLD_NOW (즉시 심볼 해석) | RTLD_LAZY (호출 시점까지 지연)
//       | RTLD_GLOBAL (로드된 심볼을 전역으로)

// 심볼 주소 획득
void *dlsym(void *handle, char *symbol);

// 공유 라이브러리 언로드
int dlclose(void *handle);

// 마지막 오류 메시지
const char *dlerror(void);
```

**사용 예시:**
```c
void *handle;
void (*addvec)(int *, int *, int *, int);

handle = dlopen("./libvector.so", RTLD_LAZY);
if (!handle) { fprintf(stderr, "%s\n", dlerror()); exit(1); }

addvec = dlsym(handle, "addvec");
if (dlerror() != NULL) { exit(1); }

addvec(x, y, z, 2);  // 일반 함수처럼 호출

dlclose(handle);
```

**컴파일:**
```bash
gcc -rdynamic -o prog2r dll.c -ldl
```

**활용 사례:**
- **소프트웨어 배포 업데이트**: Windows DLL처럼 `.so` 교체만으로 기능 업데이트
- **고성능 웹 서버**: CGI 대신 동적 로딩으로 동적 콘텐츠 처리 (fork/execve 오버헤드 제거, 함수 캐시 재사용)

---

## 7.12 Position-Independent Code (PIC)

여러 프로세스가 공유 라이브러리의 동일한 코드 복사본을 메모리에서 공유하려면, 그 코드가 링커 수정 없이 어느 주소에도 로드 가능해야 한다. 이것이 **PIC(위치 독립 코드)** 이다.

```bash
gcc -fpic -shared -o libvector.so ...
```

### GOT (Global Offset Table) - 전역 데이터 참조

**핵심 관찰:** 오브젝트 모듈이 메모리 어디에 로드되든, 코드 세그먼트와 데이터 세그먼트 사이의 거리는 항상 동일하다.

```
데이터 세그먼트 (Data Segment)
┌────────────────────────────────┐
│  GOT (Global Offset Table)     │
│  GOT[0]: …                     │
│  GOT[1]: …                     │
│  GOT[2]: …                     │
│  GOT[3]: &addcnt ←──────────┐ │  (동적 링커가 채움)
└────────────────────────────────┘
     ↑                         │
  고정 거리                     │ 런타임 상수 오프셋
  (런타임 상수)                 │
코드 세그먼트 (Code Segment)    │
┌────────────────────────────────┐
│  addvec:                       │
│    mov 0x2008b9(%rip),%rax  # %rax = GOT[3] = &addcnt
│    addl $0x1,(%rax)        # addcnt++
└────────────────────────────────┘
```

GOT 엔트리마다 재배치 레코드 생성 → 동적 링커가 로드 타임에 절대 주소로 채움

### PLT (Procedure Linkage Table) - 외부 함수 호출과 Lazy Binding

동적 링크된 함수의 주소는 런타임까지 알 수 없다. GNU는 **lazy binding**으로 첫 호출 시점까지 주소 해석을 지연한다 (대부분의 공유 라이브러리 함수는 실제로 호출되지 않을 수 있으므로).

**GOT + PLT 협동 메커니즘:**

```
GOT (Data Segment):
  GOT[0]: .dynamic 섹션 주소
  GOT[1]: reloc 엔트리 주소  
  GOT[2]: 동적 링커 진입점
  GOT[4]: 초기값 = PLT[2]+6  (addvec용)
  GOT[5]: 초기값 = PLT[3]+6  (printf용)

PLT (Code Segment):
  PLT[0]: pushq *GOT[1]      # 동적 링커 호출
          jmpq  *GOT[2]
  PLT[2]: jmpq  *GOT[4]      # addvec PLT 엔트리
          pushq $0x1          # addvec ID
          jmpq  PLT[0]
```

**첫 번째 addvec() 호출:**
```
1. main: callq PLT[2]
2. PLT[2]: jmpq *GOT[4]  → GOT[4]가 PLT[2]+6 가리킴 → fallthrough
3. PLT[2]+6: pushq $0x1  (addvec ID)
4. jmpq PLT[0]           → 동적 링커 호출
5. 동적 링커: addvec 주소 계산 → GOT[4] = &addvec으로 패치
6. addvec 실행
```

**이후 addvec() 호출:**
```
1. main: callq PLT[2]
2. PLT[2]: jmpq *GOT[4]  → GOT[4]가 &addvec 가리킴
3. 직접 addvec으로 점프  (간접 분기 1회)
```

첫 호출 이후에는 PLT 분기 1회 + GOT 메모리 참조 1회의 오버헤드만 존재한다.

---

## 7.13 Library Interpositioning

링커가 지원하는 강력한 기법: 공유 라이브러리 함수 호출을 가로채고 사용자 코드 실행. 사용 사례: 호출 횟수 추적, 입출력 검사, 함수 대체.

**원리:** 타깃 함수와 동일한 프로토타입의 wrapper 함수를 만들어 시스템을 속인다.

### 7.13.1 컴파일 타임 인터포지셔닝 (C 전처리기)

```c
// malloc.h (로컬)
#define malloc(size) mymalloc(size)  // 전처리기가 모든 malloc → mymalloc으로 교체
#define free(ptr)   myfree(ptr)

// mymalloc.c
void *mymalloc(size_t size) {
    void *ptr = malloc(size);   // 시스템 malloc 호출
    printf("malloc(%d)=%p\n", (int)size, ptr);
    return ptr;
}
```

```bash
gcc -DCOMPILETIME -c mymalloc.c
gcc -I. -o intc int.c mymalloc.o   # -I.로 로컬 malloc.h를 먼저 검색
```

### 7.13.2 링크 타임 인터포지셔닝 (--wrap 플래그)

```c
// 링커가 malloc → __wrap_malloc, __real_malloc → malloc으로 해석
void *__wrap_malloc(size_t size) {
    void *ptr = __real_malloc(size);  // 실제 libc malloc
    printf("malloc(%d) = %p\n", (int)size, ptr);
    return ptr;
}
```

```bash
gcc -DLINKTIME -c mymalloc.c
gcc -c int.c
gcc -Wl,--wrap,malloc -Wl,--wrap,free -o intl int.o mymalloc.o
```

### 7.13.3 런타임 인터포지셔닝 (LD_PRELOAD)

소스나 오브젝트 파일 없이 실행 파일만으로 인터포지셔닝 가능. 동적 링커의 `LD_PRELOAD` 환경 변수 활용.

```c
// mymalloc.c (런타임 버전)
void *malloc(size_t size) {
    // dlsym으로 다음 라이브러리의 malloc 주소 가져오기
    void *(*mallocp)(size_t) = dlsym(RTLD_NEXT, "malloc");
    char *ptr = mallocp(size);   // 실제 libc malloc 호출
    printf("malloc(%d) = %p\n", (int)size, ptr);
    return ptr;
}
```

```bash
gcc -DRUNTIME -shared -fpic -o mymalloc.so mymalloc.c -ldl
gcc -o intr int.c

# bash
LD_PRELOAD="./mymalloc.so" ./intr
# 임의 실행 파일에도 적용 가능!
LD_PRELOAD="./mymalloc.so" /usr/bin/uptime
```

**세 가지 인터포지셔닝 비교:**
| 방식 | 필요한 것 | 적용 범위 |
|------|---------|---------|
| 컴파일 타임 | 소스 파일 | 단일 프로그램 |
| 링크 타임 | 재배치 오브젝트 파일 | 단일 프로그램 |
| 런타임 | 실행 파일만 | **모든 실행 파일** |

---

## 7.14 Object File 조작 도구들 (GNU binutils)

| 도구 | 기능 |
|------|------|
| `ar` | 정적 라이브러리 생성/관리 (insert, delete, list, extract) |
| `strings` | 오브젝트 파일 내 출력 가능한 문자열 목록 |
| `strip` | 오브젝트 파일에서 심볼 테이블 정보 제거 |
| `nm` | 오브젝트 파일의 심볼 테이블에 정의된 심볼 목록 |
| `size` | 오브젝트 파일의 섹션별 이름·크기 목록 |
| `readelf` | ELF 헤더를 포함한 오브젝트 파일 전체 구조 표시 (`size` + `nm` 포함) |
| `objdump` | 오브젝트 파일의 모든 정보 표시; `.text` 섹션 역어셈블 가장 유용 |
| `ldd` | 실행 파일이 런타임에 필요로 하는 공유 라이브러리 목록 |

```bash
# 유용한 명령어 예시
nm -g libvector.a          # 라이브러리 전역 심볼 확인
objdump -d main.o          # 역어셈블
readelf -s main.o           # 심볼 테이블
readelf -l prog             # 세그먼트 헤더 (프로그램 헤더 테이블)
ldd prog2l                  # 공유 라이브러리 의존성
```

---

## 핵심 개념 정리

```
오브젝트 파일 종류:
  .o (재배치 가능) → ld → 실행 파일
  .a (정적 라이브러리, ar 아카이브)
  .so (공유 라이브러리, PIC)

심볼 해석:
  Strong (초기화) > Weak (미초기화) [Rule 2]
  Strong 중복 → 오류 [Rule 1]
  Weak 중복 → 임의 선택 (위험!) [Rule 3]

재배치 타입:
  R_X86_64_PC32: *ref = ADDR(sym) + addend - refaddr  (call)
  R_X86_64_32:   *ref = ADDR(sym) + addend             (mov imm)

PIC 핵심:
  GOT: 전역 변수 주소 간접 참조 (데이터 세그먼트)
  PLT: 외부 함수 호출 lazy binding (코드 세그먼트)
  코드/데이터 세그먼트 간 거리 = 런타임 상수 → PC-상대 주소 활용 가능

런타임 메모리 구조 (낮은 주소 → 높은 주소):
  0x400000: 코드 세그먼트 → 데이터 세그먼트 → 힙(↑) ... 스택(↓) ... 커널
```

---

## 요약

| 주제 | 핵심 |
|------|------|
| 링킹 단계 | 컴파일 타임(ld) → 로드 타임(동적 링커) → 런타임(dlopen) |
| ELF 섹션 | .text/.rodata/.data/.bss/.symtab/.rel.text/.rel.data/.strtab |
| 심볼 해석 | Strong/Weak 규칙, 조용한 중복 심볼 버그 주의 |
| 정적 라이브러리 | `.a` 아카이브, 왼쪽→오른쪽 스캔, 순서 중요 |
| 재배치 | PC-상대(PC32) / 절대(32) 두 타입, 수식으로 주소 계산 |
| 동적 링킹 | `.so` 공유 라이브러리, 메모리/디스크 공유 |
| PIC | GOT(전역 변수) + PLT(함수) lazy binding |
| 인터포지셔닝 | 컴파일(#define) → 링크(--wrap) → 런타임(LD_PRELOAD) |
