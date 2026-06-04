# Chapter 3: Machine-Level Representation of Programs

> x86-64 어셈블리 언어와 컴파일러 생성 코드를 통해 C 프로그램이 기계 수준에서 어떻게 표현되는지 탐구한다.  
> 레지스터 구조, 메모리 주소 지정, 제어 흐름, 프로시저 호출 규약, 데이터 구조의 기계어 표현, 버퍼 오버플로우 취약점을 심도 있게 다룬다.

---

## 3.1 역사적 배경 (Historical Perspective)

Intel x86 아키텍처의 진화:

| 프로세서 | 연도 | 특징 |
|---------|------|------|
| 8086 | 1978 | 16비트, 1MB 주소공간, DOS 기반 |
| 386 | 1985 | 32비트(IA32), 4GB 주소공간, 평탄 주소 모델 |
| Pentium 4E | 2004 | x86-64 지원(EM64T), 64비트 주소 |
| Core 2 | 2006 | 멀티코어 |
| Core i7 Sandy Bridge | 2011 | AVX 부동소수점, 하이퍼스레딩 |

**무어의 법칙(Moore's Law)** 에 따른 누적 확장의 결과로 x86-64는 레거시 모드와 64비트 모드를 모두 지원하는 복잡한 ISA가 되었다. 컴파일러가 이 복잡성을 흡수하므로 프로그래머는 고수준 언어만으로 활용 가능하지만, 어셈블리 이해는 성능 최적화·보안·디버깅에 필수적이다.

---

## 3.2 프로그램 인코딩 (Program Encodings)

### 머신 코드 (Machine Code)

컴파일러는 C 소스를 다음 추상화 계층을 거쳐 변환한다:

```
C source (.c)
    ↓ gcc -Og -S   (전처리 + 컴파일)
Assembly (.s)
    ↓ as            (어셈블)
Object file (.o)   ← 재배치 가능 바이너리
    ↓ ld            (링킹)
Executable
```

**머신 코드가 은닉하는 추상화:**
- 조건 코드(condition codes) — CF, ZF, SF, OF 레지스터
- 부동소수점 레지스터 (XMM/YMM)
- 메모리: 가상 주소 공간 (연속된 바이트 배열로 인식)

```bash
# 어셈블리 생성
gcc -Og -S mstore.c
# 역어셈블
objdump -d mstore.o
```

역어셈블 결과 예시 (`multstore` 함수):
```asm
0000000000000000 <multstore>:
   0: 53                   push   %rbx
   1: 48 89 d3             mov    %rdx,%rbx
   4: e8 00 00 00 00       callq  9 <multstore+0x9>
   9: 48 89 03             mov    %rax,(%rbx)
   c: 5b                   pop    %rbx
   d: c3                   retq
```

- x86-64 명령어 길이: 1~15바이트 (가변 길이)
- 역어셈블러는 바이트 패턴만으로 명령어 경계와 종류를 결정

---

## 3.3 데이터 포맷 (Data Formats)

Intel은 16비트 시절부터 "word = 2바이트" 용어를 사용한다:

| C 선언 | Intel 명칭 | 크기(바이트) | 어셈블리 접미사 |
|--------|-----------|:-----------:|:-------------:|
| `char` | Byte | 1 | `b` |
| `short` | Word | 2 | `w` |
| `int` | Double word | 4 | `l` |
| `long` | Quad word | 8 | `q` |
| `char *` | Quad word | 8 | `q` |
| `float` | Single precision | 4 | `s` |
| `double` | Double precision | 8 | `l` |

> 어셈블리 접미사 `l`은 정수(double word)와 부동소수점(double)에 모두 사용되나 피연산자 종류로 구분된다.

---

## 3.4 정보 접근 (Accessing Information)

### 3.4.1 x86-64 정수 레지스터

16개 범용 레지스터 (64비트):

```
63      31      15    8  7    0
%rax   %eax   %ax   %ah %al   ← 반환값
%rbx   %ebx   %bx   %bh %bl   ← 칼리지-저장
%rcx   %ecx   %cx   %ch %cl   ← 4번째 인자
%rdx   %edx   %dx   %dh %dl   ← 3번째 인자
%rsi   %esi   %si       %sil  ← 2번째 인자
%rdi   %edi   %di       %dil  ← 1번째 인자
%rbp   %ebp   %bp       %bpl  ← 칼리지-저장 / 프레임 포인터
%rsp   %esp   %sp       %spl  ← 스택 포인터
%r8    %r8d   %r8w      %r8b  ← 5번째 인자
%r9    %r9d   %r9w      %r9b  ← 6번째 인자
%r10   %r10d  %r10w     %r10b ← 호출자-저장
%r11   %r11d  %r11w     %r11b ← 호출자-저장
%r12   %r12d  %r12w     %r12b ← 칼리지-저장
%r13   %r13d  %r13w     %r13b ← 칼리지-저장
%r14   %r14d  %r14w     %r14b ← 칼리지-저장
%r15   %r15d  %r15w     %r15b ← 칼리지-저장
```

**32비트 연산 주의사항**: `%eax`에 값을 쓰면 상위 32비트(%rax의 bits[63:32])가 **자동으로 0으로 확장**된다. 반면 `%ax` (16비트)나 `%al` (8비트)에 쓰면 나머지 비트가 보존된다.

### 3.4.2 피연산자 형식 (Operand Specifiers)

| 형식 | 표기 | 피연산자 값 | 이름 |
|------|------|-----------|------|
| 즉시값 | `$Imm` | Imm | 즉시값 (Immediate) |
| 레지스터 | `r_a` | R[r_a] | 레지스터 |
| 메모리 (절대) | `Imm` | M[Imm] | 절대 (Absolute) |
| 메모리 (간접) | `(r_a)` | M[R[r_a]] | 간접 (Indirect) |
| 메모리 (기저+변위) | `Imm(r_b)` | M[Imm + R[r_b]] | 기저+변위 |
| 인덱스 | `(r_b, r_i)` | M[R[r_b] + R[r_i]] | 인덱스 |
| 스케일드 인덱스 | `Imm(r_b, r_i, s)` | M[Imm + R[r_b] + R[r_i]·s] | 스케일드 인덱스 |

> `s`(스케일 인수)는 1, 2, 4, 8 중 하나. 배열 인덱싱 최적화에 사용.

예시:
```asm
movq  (%rdi,%rax,4), %rdx   # rdx = M[rdi + rax*4]
leaq  (%rax,%rax,2), %rax   # rax = rax + rax*2 = rax*3
```

### 3.4.3 데이터 이동 명령어 (Data Movement Instructions)

**MOV 계열**:

| 명령어 | 효과 | 설명 |
|--------|------|------|
| `movb S, D` | D ← S | 1바이트 이동 |
| `movw S, D` | D ← S | 2바이트 이동 |
| `movl S, D` | D ← S | 4바이트 이동 (32→64비트 0확장) |
| `movq S, D` | D ← S | 8바이트 이동 |
| `movabsq I, R` | R ← I | 64비트 즉시값 이동 (레지스터만 가능) |

**MOVZ (zero-extending)**:

| 명령어 | 효과 |
|--------|------|
| `movzbw` | 바이트 → 워드 (0확장) |
| `movzbl` | 바이트 → 더블워드 (0확장) |
| `movzwl` | 워드 → 더블워드 (0확장) |
| `movzbq` | 바이트 → 쿼드워드 (0확장) |
| `movzwq` | 워드 → 쿼드워드 (0확장) |

> `movzlq`는 존재하지 않는다: `movl`이 자동으로 상위 32비트를 0으로 채우기 때문.

**MOVS (sign-extending)**:

| 명령어 | 효과 |
|--------|------|
| `movsbw` | 바이트 → 워드 (부호 확장) |
| `movsbl` | 바이트 → 더블워드 (부호 확장) |
| `movswl` | 워드 → 더블워드 (부호 확장) |
| `movsbq` | 바이트 → 쿼드워드 (부호 확장) |
| `movswq` | 워드 → 쿼드워드 (부호 확장) |
| `movslq` | 더블워드 → 쿼드워드 (부호 확장) |
| `cltq` | `%eax` → `%rax` (부호 확장), 인자 없음 |

### 3.4.4 스택 연산

스택은 높은 주소 → 낮은 주소 방향으로 성장(`%rsp`가 최상단을 가리킴):

```asm
pushq %rbp   # %rsp -= 8;  M[%rsp] = %rbp
popq  %rbp   # %rbp = M[%rsp];  %rsp += 8
```

---

## 3.5 산술 및 논리 연산 (Arithmetic and Logical Operations)

### 3.5.1 Load Effective Address (LEA)

```asm
leaq  Imm(rb, ri, s), rd    # rd = Imm + rb + ri*s
```

`MOV`와 달리 **메모리를 접근하지 않고** 주소 계산 결과만 레지스터에 저장한다. 곱셈 없이 간단한 산술에 자주 활용된다:

```asm
leaq (%rdi,%rdi,2), %rax   # rax = x + x*2 = 3x
salq $2, %rax              # rax <<= 2  →  rax = 12x
```

### 3.5.2 단항·이항 연산

| 명령어 | 효과 | 설명 |
|--------|------|------|
| `inc D` | D++ | 증가 |
| `dec D` | D-- | 감소 |
| `neg D` | D = -D | 부정 |
| `not D` | D = ~D | 비트 반전 |
| `add S, D` | D += S | 덧셈 |
| `sub S, D` | D -= S | 뺄셈 |
| `imul S, D` | D *= S | 부호 곱셈 |
| `xor S, D` | D ^= S | XOR |
| `or S, D` | D \|= S | OR |
| `and S, D` | D &= S | AND |

### 3.5.3 시프트 연산

```asm
salq $4, %rax    # rax <<= 4  (left shift = shl)
sarq $2, %rax    # rax >>= 2  (arithmetic right shift, 부호 보존)
shrq $2, %rax    # rax >>= 2  (logical right shift, 0 채움)
```

시프트 양은 즉시값 또는 `%cl` 레지스터(하위 n비트만 사용, 64비트 연산은 하위 6비트).

### 3.5.4 특수 산술 연산

**128비트 곱셈**:

```asm
imulq %rsi    # %rdx:%rax = %rax * %rsi  (부호 있는 128비트)
mulq  %rsi    # %rdx:%rax = %rax * %rsi  (부호 없는 128비트)
```

**나눗셈**:

```asm
cqto          # %rdx:%rax = 부호확장(%rax) — 128비트로 확장
idivq %rcx    # %rax = %rdx:%rax / %rcx (몫)
              # %rdx = %rdx:%rax % %rcx (나머지)
```

---

## 3.6 제어 흐름 (Control)

### 3.6.1 조건 코드 (Condition Codes)

CPU의 플래그 레지스터에 마지막 산술/논리 연산 결과를 기록한다:

| 플래그 | 이름 | 의미 |
|--------|------|------|
| `CF` | Carry Flag | 최상위 비트 자리올림(Carry) — 부호 없는 오버플로우 감지 |
| `ZF` | Zero Flag | 결과 = 0 |
| `SF` | Sign Flag | 결과 < 0 (MSB = 1) |
| `OF` | Overflow Flag | 부호 있는 오버플로우 |

**비교·테스트 명령어** (결과를 저장하지 않고 플래그만 설정):

```asm
cmpq  %rsi, %rdi   # rdi - rsi 계산 → 플래그 설정
testq %rax, %rax   # rax & rax  → ZF: rax==0 판단에 사용
```

### 3.6.2 SET 명령어

조건 코드를 읽어 바이트 레지스터에 0 또는 1 저장:

| 명령어 | 조건 | 설명 |
|--------|------|------|
| `sete` / `setz` | ZF | 같음 |
| `setne` / `setnz` | ~ZF | 다름 |
| `sets` | SF | 음수 |
| `setns` | ~SF | 양수 |
| `setg` / `setnle` | ~(SF^OF) & ~ZF | 부호 있는 `>` |
| `setge` / `setnl` | ~(SF^OF) | 부호 있는 `>=` |
| `setl` / `setnge` | SF^OF | 부호 있는 `<` |
| `setle` / `setng` | (SF^OF) \| ZF | 부호 있는 `<=` |
| `seta` / `setnbe` | ~CF & ~ZF | 부호 없는 `>` |
| `setb` / `setnae` | CF | 부호 없는 `<` |

```c
// int comp(long a, long b) { return a < b; }
comp:
    cmpq  %rsi, %rdi    # a - b
    setl  %al           # al = (a < b) ? 1 : 0
    movzbl %al, %eax    # 상위 비트 0으로 채움
    ret
```

### 3.6.3 JMP 명령어

**직접 점프 (Direct)**: 레이블로 인코딩, PC-상대 주소 사용

```asm
jmp   .L2        # 무조건 점프
je    .L4        # ZF=1이면 점프
jne   .L4        # ZF=0이면 점프
jg    .L4        # 부호 있는 >
jge   .L4        # 부호 있는 >=
jl    .L4        # 부호 있는 <
jle   .L4        # 부호 있는 <=
ja    .L4        # 부호 없는 >
jb    .L4        # 부호 없는 <
```

**간접 점프 (Indirect)**:

```asm
jmp *%rax          # rax가 가리키는 주소로 점프
jmp *(%rax)        # M[rax]가 가리키는 주소로 점프 (점프 테이블)
```

**PC-상대(PC-relative) 인코딩**: 목적지 = 현재 PC + 변위(displacement). 오브젝트 파일이 메모리의 어느 위치에 로드되더라도 정상 동작 (위치 독립 코드).

### 3.6.4 조건부 이동 (Conditional Move)

```asm
cmovge %rsi, %rax   # SF==OF이면 rax = rsi (부호 있는 >=)
```

전통적인 조건부 점프(`jxx`) 대비 파이프라인 분기 예측 실패 패널티가 없다. 컴파일러는 두 피연산자를 미리 계산한 뒤 조건에 따라 선택한다:

```c
// C:  result = x < y ? x - y : y - x;
// 컴파일러가 cmov를 사용하면:
movq  %rdi, %rax    # rax = x
subq  %rsi, %rax    # rax = x - y
movq  %rsi, %rdx    # rdx = y
subq  %rdi, %rdx    # rdx = y - x
cmpq  %rsi, %rdi    # x - y → 플래그
cmovge %rdx, %rax   # x >= y이면 rax = y - x
```

> **cmov를 사용하지 않는 경우**: 사이드이펙트가 있는 연산, NULL 역참조 가능성이 있는 식, 매우 비싼 계산(한쪽만 필요할 때).

### 3.6.5 루프 (Loops)

컴파일러는 모든 루프를 `do-while` 형태로 변환 후 최적화한다.

**do-while → 어셈블리 패턴**:

```asm
loop_body:
    ...
    jne  loop_body   # 조건 만족 시 반복
```

**while 루프 (jump-to-middle 패턴)**:

```asm
    jmp  test        # 처음부터 조건 검사
loop:
    ...body...
test:
    cmpq ...
    jl   loop
```

**for 루프**: `while`과 동일한 패턴으로 변환됨.

### 3.6.6 스위치 문 (Switch)과 점프 테이블

case 개수가 많고 값이 밀집된 경우 컴파일러는 **점프 테이블(jump table)** 을 생성한다 — O(1) 분기:

```asm
# switch(n) with cases 100..106
leaq  switch_table(%rip), %rdx    # 점프 테이블 기저 주소
movslq %esi, %rsi                  # n (32→64비트)
subq  $100, %rsi                   # n -= 100 (기저값 정규화)
cmpq  $6, %rsi                     # n > 6이면 default
ja    .Ldefault
jmp   *(%rdx,%rsi,8)               # 테이블에서 주소 읽어 점프

.section .rodata
switch_table:
    .quad .Lcase_100
    .quad .Lcase_101
    ...
```

---

## 3.7 프로시저 (Procedures)

### 3.7.1 런타임 스택 구조

x86-64 스택 프레임 레이아웃 (높은 주소 → 낮은 주소):

```
  Higher addresses
  ┌──────────────────┐
  │  Caller's frame  │
  │  ...             │
  │  7번째+ 인자     │  (호출자가 push, 역순 배치)
  │  Return address  │  ← call 명령어가 push
  ├──────────────────┤ ← %rsp (함수 진입 직후)
  │  저장된 %rbp     │  (frame pointer 사용 시)
  │  Callee-saved    │  %rbx, %r12~%r15
  │  registers       │
  │  Local variables │
  │  Argument build  │  7번째+ 인자 준비 공간
  └──────────────────┘ ← %rsp (스택 할당 후)
```

### 3.7.2 제어 전달 (Call/Ret)

```asm
callq  <label>   # push %rip(next); jmp label
retq             # pop %rip         (M[%rsp] → %rip; %rsp += 8)
```

### 3.7.3 인자 전달 (Data Transfer)

**정수/포인터 인자**: 레지스터 순서로 전달

| 인자 번호 | 64비트 | 32비트 | 16비트 | 8비트 |
|---------|--------|--------|--------|-------|
| 1 | `%rdi` | `%edi` | `%di` | `%dil` |
| 2 | `%rsi` | `%esi` | `%si` | `%sil` |
| 3 | `%rdx` | `%edx` | `%dx` | `%dl` |
| 4 | `%rcx` | `%ecx` | `%cx` | `%cl` |
| 5 | `%r8` | `%r8d` | `%r8w` | `%r8b` |
| 6 | `%r9` | `%r9d` | `%r9w` | `%r9b` |
| 7+ | 스택 | | | |

**반환값**: `%rax` (64비트) / `%eax` (32비트)

7번째 이상 인자는 **역순**으로 스택에 push됨. 스택 상에서 `8(%rsp)`가 7번째 인자, `16(%rsp)`가 8번째 인자.

### 3.7.4 지역 저장소 (Local Storage)

스택에 지역 변수를 저장해야 하는 경우:
- `&` 연산자로 주소를 취하는 변수
- 레지스터보다 지역 변수가 많을 때
- `struct` / `array` 타입 지역 변수

### 3.7.5 저장 레지스터 규약 (Register Saving Conventions)

| 구분 | 레지스터 | 책임 |
|------|---------|------|
| **호출자-저장 (Caller-saved)** | `%rax`, `%rcx`, `%rdx`, `%rsi`, `%rdi`, `%r8`~`%r11` | 호출자가 call 전 저장 필요 |
| **피호출자-저장 (Callee-saved)** | `%rbx`, `%rbp`, `%r12`~`%r15` | 피호출자가 반환 전 복원 |
| **특수** | `%rsp` | 스택 포인터, 반환 시 원상복구 |

```asm
# callee-saved 레지스터 사용 패턴
P:
    pushq  %rbp          # 저장
    pushq  %rbx
    ...
    popq   %rbx          # 복원 (역순)
    popq   %rbp
    ret
```

### 3.7.6 재귀 프로시저 (Recursive Procedures)

재귀 호출은 일반 함수 호출과 동일한 메커니즘. 각 호출마다 독립된 스택 프레임이 생성되어 지역 변수와 반환 주소가 보관된다.

```c
// 재귀 팩토리얼
long rfact(long n) {
    if (n <= 1) return 1;
    return n * rfact(n - 1);
}
```

```asm
rfact:
    pushq  %rbx
    movq   %rdi, %rbx    # 칼리지-저장 레지스터에 n 보관
    movl   $1, %eax
    cmpq   $1, %rdi
    jle    .L2
    leaq   -1(%rdi), %rdi
    call   rfact          # 재귀 호출
    imulq  %rbx, %rax    # n * rfact(n-1)
.L2:
    popq   %rbx
    ret
```

---

## 3.8 배열 할당과 접근 (Array Allocation and Access)

### 3.8.1 기본 원리

`T A[N]` 선언 시 연속된 `L * N` 바이트 (L = `sizeof(T)`) 할당.  
원소 `A[i]`의 주소 = `x_A + L * i`

```asm
# int A[5]; A의 기저 주소 = %rdx, i = %rcx
movl (%rdx,%rcx,4), %eax   # eax = A[i]
```

### 3.8.2 포인터 산술

| 표현식 | 주소 | 값 |
|--------|------|-----|
| `E` | `x_E` | 배열 주소 |
| `E[0]` | `x_E` | E[0] |
| `E[i]` | `x_E + L*i` | E[i] |
| `&E[2]` | `x_E + 2L` | 포인터 |
| `E + i - 1` | `x_E + L*(i-1)` | 포인터 |
| `*(E + i - 3)` | `x_E + L*(i-3)` | 값 |
| `&E[i] - E` | `i` | 정수 |

### 3.8.3 중첩 배열 (Nested Arrays)

`T A[R][C]` — 행 우선(Row-major) 배열:

```
A[i][j] 주소 = x_A + L * (C * i + j)
```

```asm
# int A[5][3]; i=%rsi, j=%rdi
leaq  (%rsi,%rsi,2), %rax   # rax = 3i
leaq  (%rdi,%rax), %rax     # rax = 3i + j
movl  A(,%rax,4), %eax      # eax = M[A + 4*(3i+j)]
```

---

## 3.9 이종 데이터 구조 (Heterogeneous Data Structures)

### 3.9.1 구조체 (Structures)

`struct`의 각 필드는 연속된 메모리에 배치. 컴파일러가 오프셋을 계산하여 변위 주소 지정으로 접근한다.

```c
struct rec {
    int   i;    // offset 0
    int   j;    // offset 4
    int   a[2]; // offset 8 (a[0]=8, a[1]=12)
    int  *p;    // offset 16
};              // total size = 24 bytes
```

```asm
# r = struct rec * (주소 = %rdi)
# r->i = val → movl %eax, (%rdi)
# r->a[i]   → movl (%rdi,%rsi,4), %eax  (단, %rdi += 8)
# r->p       → movq 16(%rdi), %rax
```

### 3.9.2 유니온 (Unions)

모든 필드가 같은 메모리 영역 공유. 크기 = 가장 큰 필드의 크기.

```c
union U3 {
    char   c;     // offset 0
    int    i[2];  // offset 0
    double v;     // offset 0
};                // size = 8 bytes
```

**활용 패턴 — 메모리 절약**:

```c
// 리프 노드는 data[2], 내부 노드는 left/right 포인터 사용
union node_u {
    struct {
        union node_u *left;
        union node_u *right;
    } internal;
    double data[2];
};
// struct node_s는 32바이트 필요; union node_u는 16바이트
```

**활용 패턴 — 타입 해석 전환**:

```c
// double의 비트 패턴을 unsigned long으로 읽기
unsigned long double_to_bits(double d) {
    union { double d; unsigned long u; } temp;
    temp.d = d;
    return temp.u;  // undefined behavior in C, but common in practice
}
```

### 3.9.3 데이터 정렬 (Data Alignment)

x86-64 정렬 규칙: **K바이트 타입은 K의 배수 주소**에 배치.

| K | 타입 |
|---|------|
| 1 | `char` |
| 2 | `short` |
| 4 | `int`, `float` |
| 8 | `long`, `double`, `char *` |

컴파일러는 구조체 필드 사이에 **패딩(padding)** 을 삽입한다:

```c
struct S1 {
    int  i;   // offset 0
    char c;   // offset 4
    // [3 bytes padding]
    int  j;   // offset 8  (4의 배수)
};            // size = 12

// 필드 재배열로 패딩 최소화:
struct S2 {
    int  i;   // offset 0
    int  j;   // offset 4
    char c;   // offset 8
    // [3 bytes padding]
};            // size = 12 (동일; tail padding 필요)

// 최적화:
struct S3 {
    char c;   // offset 0
    // [3 bytes padding]
    int  i;   // offset 4
    int  j;   // offset 8
};            // size = 12
```

> 구조체 크기는 최대 정렬 요구사항의 배수가 되어야 배열에서도 정렬이 유지된다.

**SSE/AVX 강제 정렬**: 16-바이트 정렬 필수(`alloca`, `malloc` 등 모두 16바이트 경계 반환, 대부분 스택 프레임도 16바이트 경계 유지).

---

## 3.10 제어와 데이터의 결합 (Combining Control and Data)

### 3.10.1 포인터 이해

기계어 수준에서 포인터의 핵심 원칙:

- **모든 포인터는 연관된 타입을 가진다** — 기계어에는 타입 정보가 없음; C의 추상화
- **포인터 값 = 주소** — `NULL(0)`은 어디도 가리키지 않음
- `&` 연산자는 주로 `leaq`로 구현됨
- `*` 역참조는 메모리 참조 명령어로 구현됨
- **배열과 포인터의 관계**: `a[i]` ≡ `*(a + i)`, 오프셋 스케일링은 타입 크기에 따름
- **함수 포인터**: 값 = 함수의 첫 번째 명령어 주소

```c
int (*fp)(int, int *);  // fp는 (int, int*) → int 함수의 포인터
fp = fun;               // 함수 주소 저장
result = fp(3, &y);     // 간접 호출 → jmp *fp (어셈블리)
```

### 3.10.2 GDB 활용

```
(gdb) break multstore        # 함수 진입점에 브레이크포인트
(gdb) break *0x400540        # 특정 주소에 브레이크포인트
(gdb) disas                  # 현재 함수 역어셈블
(gdb) stepi                  # 명령어 단위 실행
(gdb) print/x $rax           # 레지스터 값 출력 (hex)
(gdb) x/2g 0x7fffffffe818    # 메모리 2개 quad word 출력
(gdb) info registers         # 모든 레지스터 값
```

### 3.10.3 버퍼 오버플로우 (Buffer Overflow)

C는 배열 경계 검사가 없으므로 스택에 할당된 버퍼를 초과 쓰면 반환 주소가 덮어써진다:

```c
char *gets(char *s) {    // 안전하지 않은 표준 라이브러리 함수!
    int c;
    char *dest = s;
    while ((c = getchar()) != '\n' && c != EOF)
        *dest++ = c;
    *dest++ = '\0';
    return s;
}

void echo() {
    char buf[8];    // 8바이트 버퍼
    gets(buf);      // 경계 검사 없음 → 위험!
    puts(buf);
}
```

```asm
echo:
    subq $24, %rsp      # 24바이트 할당 (buf = 8 + 16 여유)
    movq %rsp, %rdi
    call gets
    ...
```

스택 레이아웃:
```
  %rsp+24  [Return address]  ← 32+ 문자 입력 시 덮어쓰기!
  %rsp+8   [Unused 16 bytes]
  %rsp+0   [buf[0]~buf[7]]
```

| 입력 길이 | 손상 영역 |
|-----------|----------|
| 0~7 | 없음 |
| 9~23 | 사용되지 않는 스택 공간 |
| 24~31 | 반환 주소 |
| 32+ | 호출자의 저장 상태 |

### 3.10.4 버퍼 오버플로우 대응 기법

**① 스택 카나리 (Stack Canary / Stack Protector)**

gcc `-fstack-protector` 옵션 (현재 기본 활성화):

```asm
echo:
    subq   $24, %rsp
    movq   %fs:40, %rax      # 카나리 값 읽기 (thread-local storage)
    movq   %rax, 8(%rsp)     # 스택의 buf와 반환 주소 사이에 저장
    xorl   %eax, %eax
    movq   %rsp, %rdi
    call   gets
    ...
    movq   8(%rsp), %rax     # 카나리 읽기
    xorq   %fs:40, %rax      # 원본과 XOR 비교
    je     .L8               # 같으면 정상 반환
    call   __stack_chk_fail  # 달라졌으면 프로그램 종료
```

- `%fs:40`: 읽기 전용 세그먼트에 저장 → 공격자가 덮어쓸 수 없음
- 프로세스마다 랜덤 값 사용 → 예측 불가

**② ASLR (Address Space Layout Randomization)**

- 스택, 힙, 공유 라이브러리 등의 주소를 실행마다 랜덤 배치
- 공격자가 쉘코드(shellcode) 주입 위치를 예측하기 어려워짐
- 단, 64비트 주소 공간 중 일부 비트만 랜덤화됨 (NOP sled 공격으로 우회 가능)

**③ 실행 불가 페이지 (NX bit / No-Execute)**

- 스택, 힙 등 데이터 영역에 실행 권한 제거 (하드웨어 지원)
- 스택에 주입된 코드를 실행하려 하면 예외 발생
- 우회: Return-Oriented Programming (ROP) — 기존 코드 가젯(gadget) 조합

> 세 가지 기법을 함께 적용하면 버퍼 오버플로우 악용이 극적으로 어려워진다. 단, ROP 등 우회 기술도 존재하므로 근본적으로 안전한 코드를 작성해야 한다.

### 3.10.5 가변 크기 스택 프레임

`alloca()` 또는 가변 길이 배열(VLA) 사용 시, 컴파일러가 스택 크기를 컴파일 타임에 결정할 수 없다.  
이 경우 `%rbp` (프레임 포인터)를 사용한다:

```c
long vframe(long n, long idx, long *q) {
    long i;
    long *p[n];    // VLA: 크기가 런타임에 결정
    ...
}
```

```asm
vframe:
    pushq  %rbp            # 이전 %rbp 저장 (callee-saved)
    movq   %rsp, %rbp      # %rbp = 프레임 기저 포인터
    subq   $16, %rsp       # i를 위한 고정 공간
    leaq   22(,%rdi,8), %rax
    andq   $-16, %rax      # 16바이트 정렬
    subq   %rax, %rsp      # p[] 배열 동적 할당
    ...
    movq   -8(%rbp), %rax  # i는 %rbp 상대 주소로 접근
    ...
    leave                  # movq %rbp,%rsp; popq %rbp
    ret
```

`leave` 명령어 = `movq %rbp, %rsp; popq %rbp` — 가변 크기 프레임을 안전하게 해제.

---

## 3.11 부동소수점 코드 (Floating-Point Code)

x86-64는 **AVX2** (Advanced Vector Extensions 2) 또는 **SSE** (Streaming SIMD Extensions) 기반으로 부동소수점을 처리한다.

### 3.11.1 미디어 레지스터

16개의 YMM/XMM 레지스터:

```
%ymm0 ~ %ymm15  (각 256비트 = 32바이트)
   └── %xmm0 ~ %xmm15  (하위 128비트 = 16바이트)
```

- **스칼라 모드**: 하위 32비트(float) 또는 64비트(double) 사용
- **패킹 모드**: 전체 레지스터에 여러 값을 병렬 처리 (SIMD)

### 3.11.2 부동소수점 이동 명령어

| 명령어 | 피연산자 | 설명 |
|--------|---------|------|
| `vmovss` | M₃₂ ↔ X | 단정밀도 스칼라 이동 |
| `vmovsd` | M₆₄ ↔ X | 배정밀도 스칼라 이동 |
| `vmovaps` | X ↔ X | 정렬된 패킹 단정밀도 이동 |
| `vmovapd` | X ↔ X | 정렬된 패킹 배정밀도 이동 |

### 3.11.3 변환 명령어

```asm
# float/double → 정수 변환 (truncation)
vcvttss2si  %xmm0, %eax    # float → int (버림)
vcvttsd2si  %xmm0, %eax    # double → int (버림)
vcvttsd2siq %xmm0, %rax    # double → long (버림)

# 정수 → float/double 변환
vcvtsi2ssl  %eax, %xmm1, %xmm0   # int → float
vcvtsi2sdl  %eax, %xmm1, %xmm0   # int → double
vcvtsi2ssq  %rax, %xmm1, %xmm0   # long → float
vcvtsi2sdq  %rax, %xmm1, %xmm0   # long → double

# float ↔ double 변환
vcvtss2sd   %xmm0, %xmm0, %xmm0  # float → double
vcvtsd2ss   %xmm0, %xmm0, %xmm0  # double → float
```

### 3.11.4 프로시저에서의 부동소수점

- **부동소수점 인자**: `%xmm0` ~ `%xmm7` (순서대로)
- **반환값**: `%xmm0`
- **XMM 레지스터는 모두 호출자-저장(Caller-saved)**: 피호출자가 자유롭게 수정 가능

```c
double funct(long x, double y, long z) {
    // x: %rdi, y: %xmm0, z: %rdx
    return ...;  // 반환: %xmm0
}
```

### 3.11.5 부동소수점 산술 연산

| 명령어 | 효과 | 설명 |
|--------|------|------|
| `vaddss`/`vaddsd` | D ← S₂ + S₁ | 덧셈 |
| `vsubss`/`vsubsd` | D ← S₂ - S₁ | 뺄셈 |
| `vmulss`/`vmulsd` | D ← S₂ × S₁ | 곱셈 |
| `vdivss`/`vdivsd` | D ← S₂ / S₁ | 나눗셈 |
| `vmaxss`/`vmaxsd` | D ← max(S₂, S₁) | 최댓값 |
| `vminss`/`vminsd` | D ← min(S₂, S₁) | 최솟값 |
| `vsqrtss`/`vsqrtsd` | D ← √S₁ | 제곱근 |

**부동소수점 상수**: 즉시값(immediate) 없음 → 메모리에서 로드

```asm
# double cel2fahr(double temp) { return 1.8 * temp + 32.0; }
cel2fahr:
    vmulsd .LC2(%rip), %xmm0, %xmm0   # temp * 1.8
    vaddsd .LC3(%rip), %xmm0, %xmm0   # + 32.0
    ret
.LC2:
    .long  3435973837   # 1.8의 하위 32비트 (리틀 엔디언)
    .long  1073532108   # 1.8의 상위 32비트
.LC3:
    .long  0
    .long  1077936128   # 32.0
```

### 3.11.6 부동소수점 비교

```asm
vucomiss  %xmm1, %xmm0   # float 비교: xmm0 vs xmm1
vucomisd  %xmm1, %xmm0   # double 비교
```

| CF | ZF | PF | 의미 |
|----|----|----|------|
| 0 | 0 | 0 | S₂ > S₁ |
| 0 | 1 | 0 | S₂ = S₁ |
| 1 | 0 | 0 | S₂ < S₁ |
| 1 | 1 | 1 | Unordered (NaN 포함) |

- `PF` (Parity Flag): 피연산자 중 하나가 NaN이면 1
- `x != x`는 `x`가 NaN일 때만 참 → NaN 감지에 활용

---

## 3.12 요약

```
┌─────────────────────────────────────────────────────────────────┐
│                   x86-64 핵심 개념 정리                          │
├─────────────────┬───────────────────────────────────────────────┤
│ 레지스터        │ 16개 정수(+특수) + 16개 XMM/YMM (부동소수점)   │
├─────────────────┼───────────────────────────────────────────────┤
│ 주소 지정       │ 9가지 메모리 주소 지정 모드, 최대 Imm+Rb+Ri*s   │
├─────────────────┼───────────────────────────────────────────────┤
│ 조건 코드       │ CF, ZF, SF, OF — cmp/test로 설정, setXX로 읽기  │
├─────────────────┼───────────────────────────────────────────────┤
│ 프로시저        │ call/ret + 6개 레지스터 인자 + 반환 %rax        │
├─────────────────┼───────────────────────────────────────────────┤
│ 보안            │ 스택 카나리 + ASLR + NX bit 3중 방어            │
├─────────────────┼───────────────────────────────────────────────┤
│ 부동소수점      │ AVX/SSE XMM 레지스터, 인자 %xmm0~7            │
└─────────────────┴───────────────────────────────────────────────┘
```

### 핵심 교훈

- **컴파일러 최적화를 이해하라**: `cmov`, 루프 변환, LEA를 이용한 곱셈 등 패턴을 알면 성능 분석에 도움
- **스택 규약을 준수하라**: 16바이트 정렬, 칼리지-저장 레지스터 보존
- **버퍼 오버플로우는 기계어 수준에서만 이해된다**: C 코드만 보면 왜 오버플로우가 위험한지 알기 어렵다
- **부동소수점은 별도 레지스터 파일**: 정수 레지스터와 분리, 명시적 변환 필요
