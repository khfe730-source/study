# Chapter 4: Processor Architecture

> 프로세서 하드웨어 설계의 핵심 원리를 Y86-64라는 단순화된 ISA를 통해 탐구한다.  
> 순차(SEQ) 프로세서에서 출발하여 파이프라인(PIPE) 설계로 발전시키며, 데이터 해저드·제어 해저드를 해결하는 포워딩(forwarding)·스톨(stall)·버블(bubble) 메커니즘을 깊이 다룬다.

---

## 4.1 Y86-64 명령어 집합 아키텍처 (Instruction Set Architecture)

Y86-64는 x86-64를 단순화한 교육용 ISA. **8바이트 정수 연산만 지원**, 단순한 바이트 수준 인코딩, 제한된 주소 지정 모드.

### 4.1.1 프로그래머가 볼 수 있는 상태 (Programmer-Visible State)

```
RF:  Program Registers      CC: Condition Codes    Stat: Status
%rax  %rsp  %r8  %r12       ZF  SF  OF             AOK/HLT/ADR/INS
%rcx  %rbp  %r9  %r13
%rdx  %rsi  %r10 %r14       PC: Program Counter
%rbx  %rdi  %r11
                             DMEM: Memory (가상 주소 공간)
```

- 레지스터 15개 (`%r15` 제외 — 인코딩 단순화 목적)
- 조건 코드 3비트: ZF, SF, OF (x86-64의 CF 없음)
- 상태 코드 `Stat`: 정상/예외 구분

### 4.1.2 Y86-64 명령어 세트

```
Byte:        0       1       2~9
────────────────────────────────────────────
halt         0x00
nop          0x10
rrmovq rA,rB 0x20   rA:rB
irmovq V,rB  0x30   F:rB    V(8바이트)
rmmovq rA,D(rB) 0x40 rA:rB D(8바이트)
mrmovq D(rB),rA 0x50 rA:rB D(8바이트)
OPq rA,rB    0x6fn  rA:rB              fn: add=0,sub=1,and=2,xor=3
jXX Dest     0x7fn  Dest(8바이트)       fn: jmp=0,jle=1,jl=2,je=3,jne=4,jge=5,jg=6
cmovXX rA,rB 0x2fn  rA:rB              fn: 위와 동일 (rrmovq = cmovXX fn=0)
call Dest    0x80   Dest(8바이트)
ret          0x90
pushq rA     0xA0   rA:F
popq  rA     0xB0   rA:F
```

명령어 길이: **1~10바이트** (가변 길이). 첫 바이트 상위 4비트 = icode, 하위 4비트 = ifun.

**주요 차이점 (vs x86-64)**:
- `movq` → 4가지로 분리: `irmovq`(즉시→레지스터), `rrmovq`(레지스터→레지스터), `rmmovq`(레지스터→메모리), `mrmovq`(메모리→레지스터)
- OPq는 레지스터 피연산자만 지원 (x86-64는 메모리 피연산자도 가능)
- 메모리 주소: `D(rB)` 형식만 지원 (인덱스 레지스터나 스케일링 없음)
- `0xF` = 레지스터 없음(no register)을 표시하는 특수 ID

### 4.1.3 레지스터 ID

| ID | 레지스터 | | ID | 레지스터 |
|----|---------||----|---------|
| 0 | `%rax` | | 8 | `%r8` |
| 1 | `%rcx` | | 9 | `%r9` |
| 2 | `%rdx` | | A | `%r10` |
| 3 | `%rbx` | | B | `%r11` |
| 4 | `%rsp` | | C | `%r12` |
| 5 | `%rbp` | | D | `%r13` |
| 6 | `%rsi` | | E | `%r14` |
| 7 | `%rdi` | | F | (없음) |

### 4.1.4 예외 및 상태 코드

| 값 | 이름 | 의미 |
|----|------|------|
| 1 | AOK | 정상 실행 |
| 2 | HLT | `halt` 명령어 실행 |
| 3 | ADR | 유효하지 않은 메모리 주소 접근 |
| 4 | INS | 유효하지 않은 명령어 코드 |

Y86-64 프로세서는 AOK가 아닌 상태 코드가 발생하면 즉시 실행을 중단한다.

### 4.1.5 Y86-64 프로그램 예시

```c
long sum(long *start, long count) {
    long sum = 0;
    while (count) { sum += *start; start++; count--; }
    return sum;
}
```

```asm
# Y86-64 어셈블리
sum:
    irmovq  $8, %r8       # 상수를 먼저 레지스터에 로드 (x86-64와의 차이)
    irmovq  $1, %r9
    xorq    %rax, %rax    # sum = 0
    andq    %rsi, %rsi    # count 테스트
    jmp     test
loop:
    mrmovq  (%rdi), %r10  # *start → r10
    addq    %r10, %rax    # sum += *start
    addq    %r8,  %rdi    # start++
    subq    %r9,  %rsi    # count--
test:
    jne     loop
    ret
```

---

## 4.2 논리 설계와 HCL (Logic Design and HCL)

### 4.2.1 논리 게이트 (Logic Gates)

| 게이트 | HCL 표기 | 설명 |
|--------|---------|------|
| AND | `a && b` | 모든 입력이 1일 때 출력 1 |
| OR | `a \|\| b` | 하나라도 1이면 출력 1 |
| NOT | `!a` | 반전 |

논리 게이트는 **항상 활성(always active)**. 입력이 바뀌면 일정 지연(propagation delay) 후 출력 변경.

### 4.2.2 조합 회로 (Combinational Circuits)

**제약 조건**:
- 모든 게이트 입력은 단 하나의 소스에 연결
- 게이트 출력은 합선 불가 (single driver)
- 회로는 비순환(acyclic) — 피드백 루프 없음

HCL 예시 — 1비트 동등 비교:
```hcl
bool eq = (a && b) || (!a && !b);
```

1비트 MUX (multiplexor):
```hcl
bool out = (s && a) || (!s && b);  # s=1이면 a, s=0이면 b
```

### 4.2.3 워드 수준 회로와 HCL 정수 표현

HCL **case 표현식** (word-level MUX):
```hcl
word out = [
    s1 && s2 : A;   # s1=1, s2=1
    s1       : B;   # s1=1, s2=0
    s2       : C;   # s1=0, s2=1
    1        : D;   # default
];
```

조건은 위에서 아래로 순서대로 평가. 첫 번째로 참인 조건의 값이 선택됨.

### 4.2.4 집합 멤버십 (Set Membership)

```hcl
bool jump = icode in { IJMP, IJLE, IJL, IJE, IJNE, IJGE, IJG };
```

`in` 연산자: 신호가 집합의 임의 원소와 같으면 1.

### 4.2.5 메모리와 클로킹

**두 종류의 저장 장치**:

| 종류 | 특성 | 사용 예 |
|------|------|---------|
| **클로킹 레지스터 (Clocked register)** | 클록 상승 에지(rising edge)에만 값 업데이트 | PC, CC(조건 코드), 파이프라인 레지스터 |
| **랜덤 접근 메모리 (RAM)** | 주소로 읽기/쓰기, 쓰기는 클록 에지에 동기화 | 레지스터 파일(register file), 명령어 메모리, 데이터 메모리 |

```
클록 사이클 동작:
  클록 낮음: 조합 논리가 신호 전파 (게이트 출력이 안정화됨)
  클록 상승: 레지스터 입력 값을 캡처하여 새 상태로 저장
  클록 높음: 레지스터 출력 = 새 상태
```

**레지스터 파일 (Register File)**:
```
     srcA ──► A 포트 ──► valA
     srcB ──► B 포트 ──► valB   (읽기: 비동기/즉시)
     dstE ◄── E 포트 ◄── valE
     dstM ◄── M 포트 ◄── valM   (쓰기: 클록 에지에 동기화)
```

---

## 4.3 순차 Y86-64 구현 (SEQ)

### 4.3.1 처리 단계 (Processing Stages)

SEQ는 매 클록 사이클마다 명령어 하나를 **6단계**로 처리한다:

```
┌──────────┐
│  Fetch   │ → icode, ifun, rA, rB, valC, valP 추출 (PC에서 명령어 읽기)
├──────────┤
│  Decode  │ → valA, valB 읽기 (레지스터 파일에서 소스 피연산자 로드)
├──────────┤
│  Execute │ → valE 계산 (ALU 연산 또는 주소 계산), 조건 코드 갱신, Cnd 결정
├──────────┤
│  Memory  │ → 메모리 읽기(valM) 또는 쓰기
├──────────┤
│ Write-   │ → valE, valM을 레지스터 파일에 기록 (dstE, dstM)
│  back    │
├──────────┤
│ PC Update│ → 다음 PC 결정 (valP, valC, valM 중 선택)
└──────────┘
```

**각 명령어별 처리 요약**:

| 단계 | OPq rA, rB | mrmovq D(rB), rA | push rA | call Dest |
|------|------------|-----------------|---------|-----------|
| Fetch | icode=6, rA, rB | icode=5, rA, rB, D | icode=A, rA | icode=8, Dest |
| Decode | valA=R[rA], valB=R[rB] | valB=R[rB] | valA=R[rA], valB=R[%rsp] | valB=R[%rsp] |
| Execute | valE=valA OP valB, 조건코드 설정 | valE=D+valB | valE=valB−8 | valE=valB−8 |
| Memory | — | valM=M[valE] | M[valE]=valA | M[valE]=valP |
| Write-back | R[rB]=valE | R[rA]=valM | R[%rsp]=valE | R[%rsp]=valE |
| PC Update | valP | valP | valP | Dest |

### 4.3.2 SEQ 하드웨어 구조

```
                          newPC
                            ↑
PC ──► Instruction ──► Decode ──► Execute ──► Memory ──► Write-back
       Memory       Register    ALU        Data
       (Fetch)      File        CC←───────  Memory
          ↑                                    ↓
         icode, ifun, rA, rB, valC, valP     valM
```

**SEQ 구성 요소**:
- **명령어 메모리**: PC로 바이트 시퀀스 읽기, icode/ifun/레지스터 ID/상수 추출
- **레지스터 파일**: 2포트 읽기(srcA, srcB), 2포트 쓰기(dstE, dstM)
- **ALU**: valE 계산, 조건 코드(CC) 갱신
- **데이터 메모리**: 읽기 또는 쓰기

### 4.3.3 SEQ 타이밍

SEQ의 **4개 클로킹 상태 요소**: PC, CC, 레지스터 파일, 데이터 메모리

```
클록 주기:
  ┌─────────────────────────────────────────────┐
  │ 조합 논리 전파 (Fetch→Decode→Execute→...→PC)│
  └──────────────────────────────────────────────┘
  ↑ 클록 상승                                    ↑ 다음 클록 상승
  상태 요소에 새 값 로드
```

**원칙**: 클로킹 레지스터와 메모리는 클록 에지에만 업데이트 → 각 클록 사이클에 하나의 명령어를 원자적으로 처리.

### 4.3.4 HCL 제어 로직 예시

```hcl
# 다음 PC 선택 (PC Update stage)
word new_pc = [
    icode == ICALL         : valC;   # call → 목적지 주소
    icode == IJXX && Cnd   : valC;   # 분기 성공 → 목적지 주소
    icode == IRET          : valM;   # ret → 스택에서 읽은 반환 주소
    1                      : valP;   # 기본: 다음 순서 명령어
];

# ALU A 입력 선택
word aluA = [
    icode in { IRRMOVQ, IOPQ }       : valA;
    icode in { IIRMOVQ, IRMMOVQ, IMRMOVQ } : valC;
    icode in { ICALL, IPUSHQ }       : -8;
    icode in { IRET, IPOPQ }         : 8;
];
```

---

## 4.4 파이프라이닝의 일반 원리 (General Principles of Pipelining)

### 4.4.1 처리량과 지연 시간

**비파이프라인 시스템** (예: 조합 논리 300ps + 레지스터 20ps):
$$\text{처리량} = \frac{1}{300+20} \text{ ps} = 3.12 \text{ GIPS}$$

**3단계 파이프라인** (각 단계 100ps + 레지스터 20ps):
$$\text{처리량} = \frac{1}{100+20} \text{ ps} = 8.33 \text{ GIPS} \quad \text{(2.67× 향상)}$$
$$\text{지연(latency)} = 3 \times 120 = 360 \text{ ps} \quad \text{(비파이프라인 320ps보다 증가)}$$

> **핵심**: 파이프라이닝은 처리량(throughput)을 높이지만 개별 명령어의 지연(latency)은 늘어날 수 있다.

### 4.4.2 파이프라인 레지스터의 동작

```
       100ps      20ps      100ps      20ps      100ps      20ps
  ┌─────────┐  ┌─────┐  ┌─────────┐  ┌─────┐  ┌─────────┐  ┌─────┐
  │ Comb.   │→ │  R  │→ │ Comb.   │→ │  R  │→ │ Comb.   │→ │  R  │
  │ Logic A │  │  e  │  │ Logic B │  │  e  │  │ Logic C │  │  e  │
  └─────────┘  └──g──┘  └─────────┘  └──g──┘  └─────────┘  └──g──┘
                  ↑                      ↑                      ↑
               클록 에지에 캡처         클록 에지에 캡처         클록 에지에 캡처
```

클록 주기 동안 조합 논리가 안정화되고, 클록 상승 에지에 레지스터가 값을 캡처한다.

### 4.4.3 파이프라이닝의 한계

**① 비균일 단계 지연 (Nonuniform Partitioning)**

단계 지연이 50ps, 150ps, 100ps로 불균등하면:
- 클록 주기 = 최대 단계 지연 = 150+20 = 170ps
- 처리량 = 5.88 GIPS (균일 분할 시 8.33 GIPS 대비 손해)
- 느린 단계가 병목(bottleneck)

**② 깊은 파이프라인의 수익 감소 (Diminishing Returns of Deep Pipelining)**

파이프라인 레지스터 자체의 지연(overhead, 20ps)이 클록 주기에서 차지하는 비중 증가:
$$\text{클록 주기} = \frac{300}{k} + 20 \text{ ps (k단계 파이프라인)}$$

현대 프로세서는 15+ 단계의 깊은 파이프라인을 사용하며, 각 단계를 극도로 단순화한다.

### 4.4.4 파이프라인 해저드 (Pipeline Hazards)

파이프라이닝된 시스템에서 연속 명령어 간의 **의존성(dependency)** 이 실제 문제를 일으키는 것을 **해저드(hazard)** 라고 한다.

| 종류 | 원인 | 해결책 |
|------|------|--------|
| **데이터 해저드** | 이전 명령어가 아직 완료하지 않은 레지스터 값을 읽으려 할 때 | 포워딩, 스톨 |
| **제어 해저드** | 분기/ret 명령어의 다음 실행 주소를 알 수 없을 때 | 예측 후 버블 삽입 |

---

## 4.5 파이프라인 Y86-64 구현 (PIPE)

### 4.5.1 SEQ+ — 단계 재배열

PIPE 설계를 위해 SEQ에서 PC Update 단계를 **사이클의 시작**으로 이동 → **SEQ+**:

```
SEQ:  Fetch → Decode → Execute → Memory → Write-back → [PC Update]
SEQ+: [PC Update] → Fetch → Decode → Execute → Memory → Write-back
```

PC Update가 먼저 일어나므로 현재 PC 값은 레지스터에 보관(클로킹됨)되지 않고 조합 논리로 계산.

### 4.5.2 PIPE−: 파이프라인 레지스터 삽입

SEQ+ 각 단계 사이에 **파이프라인 레지스터** 삽입:

```
F ──── D ──── E ──── M ──── W
│      │      │      │      │
Fetch  Decode Execute Memory Write-back
```

**파이프라인 레지스터 명칭과 보관 내용**:

| 레지스터 | 위치 | 보관 내용 |
|---------|------|---------|
| `F` | Fetch 앞 | 예측된 PC 값 |
| `D` | Fetch/Decode 사이 | icode, ifun, rA, rB, valC, valP, Stat |
| `E` | Decode/Execute 사이 | valA, valB, dstE, dstM, aluA, aluB, alufun |
| `M` | Execute/Memory 사이 | valE, valA, dstE, dstM, Cnd, Stat |
| `W` | Memory/Write-back 사이 | valE, valM, dstE, dstM, Stat |

### 4.5.3 파이프라인 다이어그램

```
Time(cycle):  1    2    3    4    5    6    7    8    9
─────────────────────────────────────────────────────────
Inst 1:       F    D    E    M    W
Inst 2:            F    D    E    M    W
Inst 3:                 F    D    E    M    W
Inst 4:                      F    D    E    M    W
Inst 5:                           F    D    E    M    W
```

이상적인 경우 매 사이클마다 새 명령어가 완료 → CPI = 1.0.

### 4.5.4 다음 PC 예측 (Next PC Prediction)

파이프라인은 분기 명령어의 결과를 알기 전에 다음 명령어를 페치해야 한다.

**PIPE의 분기 예측 전략**: **항상 분기 안 함(predict not-taken)** 가정
- 조건부 점프 → 순차적으로 다음 명령어 실행 예측
- 예측 실패 시: 파이프라인에 들어온 두 명령어를 버블로 대체

**call, jmp**: 항상 목적지 주소가 알려져 있으므로 예측 가능 (predict taken).

### 4.5.5 파이프라인 해저드 처리

#### 데이터 해저드 (Data Hazards)

Decode 단계에서 레지스터 파일을 읽지만, 아직 Write-back 단계에 도달하지 않은 결과가 필요한 경우:

```
Cycle:      1    2    3    4    5    6    7
irmovq $10,%rdx  F    D    E    M    W
irmovq  $3,%rax       F    D    E    M    W
addq %rdx,%rax             F    D    E    M    W
                                 ↑
                                 %rdx는 아직 Write-back 전
                                 → 데이터 해저드!
```

**① 포워딩 (Data Forwarding / Bypassing)**

파이프라인 레지스터에서 아직 레지스터 파일에 기록되지 않은 값을 Decode 단계로 직접 전달:

```
포워딩 소스 (우선순위 순):
  e_valE  ← Execute/Memory 파이프라인 레지스터 valE
  m_valM  ← Memory/Write-back 파이프라인 레지스터 valM (메모리 읽기 결과)
  W_valE  ← Write-back 단계 valE
  W_valM  ← Write-back 단계 valM
  레지스터 파일 (포워딩 없는 경우)
```

HCL 표현:
```hcl
word d_valA = [
    D_icode in { ICALL, IJXX } : D_valP;  # call/jmp: valP 직접 사용
    d_srcA == e_dstE            : e_valE;  # Execute 단계 결과 포워딩
    d_srcA == M_dstE            : M_valE;
    d_srcA == m_dstM            : m_valM;  # 메모리 읽기 결과 포워딩
    d_srcA == W_dstE            : W_valE;
    d_srcA == W_dstM            : W_valM;
    1                           : d_rvalA; # 레지스터 파일 읽기
];
```

**② 로드-사용 해저드와 스톨 (Load/Use Hazard & Stall)**

`mrmovq`(메모리 읽기)의 결과를 다음 명령어가 즉시 사용하는 경우 포워딩만으로 불가 (메모리 읽기는 Execute 이후):

```
Cycle:      1    2    3    4    5    6    7    8
mrmovq 0(%rdi),%rax  F    D    E    M    W
addq %rax,%rbx            F    D  stall  D    E    M    W
                                ↑  ↑
                                  로드 인터록(load interlock): Decode 1사이클 지연
                              Execute에 bubble 삽입
```

**로드 인터록(Load Interlock)**: 로드 명령어가 Execute 단계에 있고 다음 명령어가 Decode에서 같은 레지스터를 읽으면 1사이클 스톨. 스톨 후 m_valM 포워딩으로 처리.

#### 제어 해저드 (Control Hazards)

**① ret 명령어**

ret은 스택에서 반환 주소를 읽어야 하므로 Memory 단계까지 실제 주소를 알 수 없음 → **3사이클 버블**:

```
Cycle:      1    2    3    4    5    6    7
ret          F    D    E    M    W
bubble            F    D    E    M    W
bubble                 F    D    E    M    W
bubble                      F    D    E    M    W
next_after_ret                        F    D    E    M    W
```

**② 잘못 예측된 분기 (Mispredicted Branch)**

not-taken으로 예측했으나 실제로 taken인 경우 → **2사이클 버블**:

```
Cycle:      1    2    3    4    5    6    7
jXX target   F    D    E    M    W
instr_after       F    D  cancel (bubble)
instr_after+1          F  cancel (bubble)
instr_at_target             F    D    E    M    W
```

Execute 단계에서 Cnd(분기 조건) 결정 후 파이프라인 앞의 두 명령어를 버블로 대체.

### 4.5.6 파이프라인 제어 로직 (Pipeline Control Logic)

특수 상황을 처리하는 제어 신호:

| 상황 | Fetch 레지스터 | Decode 레지스터 | Execute 레지스터 |
|------|--------------|---------------|----------------|
| 로드-사용 해저드 | **유지(stall)** | **유지(stall)** | **버블(bubble)** |
| ret 처리 중 | **유지(stall)** | **버블(bubble)** | 정상 |
| 잘못된 분기 예측 | 정상 | **버블(bubble)** | **버블(bubble)** |

```
┌────────────────────────────────────────────────┐
│             파이프라인 제어 조건                  │
├──────────────┬─────────────────────────────────┤
│ load_use     │ E_icode in {MRMOVQ, POPQ}         │
│              │ && E_dstM in {d_srcA, d_srcB}     │
├──────────────┼─────────────────────────────────┤
│ ret_stall    │ IRET in {D_icode, E_icode, M_icode}│
├──────────────┼─────────────────────────────────┤
│ mis_branch   │ E_icode == IJXX && !e_Cnd        │
└──────────────┴─────────────────────────────────┘
```

### 4.5.7 예외 처리 (Exception Handling)

파이프라인에서 여러 명령어가 동시에 실행 중인 상황의 예외 처리 원칙:

1. **이후 명령어의 부작용 방지**: 예외 발생 명령어가 Write-back에 도달하기 전, 파이프라인의 이후 단계(Memory, Write-back)에 있는 명령어들의 상태 변경을 금지
2. **조용한 예외(Silent Exception)**: 예외 발생 사실을 상태 코드로 파이프라인에 흘려보내고 Write-back 단계에 도달할 때 최종 처리
3. **Memory/Write-back의 Stat 체크**: Write-back 단계에서 Stat ≠ AOK이면 업데이트 중단

### 4.5.8 PIPE 성능 분석 (CPI)

$$\text{CPI} = 1.0 + lp + mp + rp$$

| 항목 | 설명 | 버블 수 |
|------|------|---------|
| $lp$ (load penalty) | 로드-사용 해저드 버블 발생 빈도 | 1사이클/발생 |
| $mp$ (misprediction penalty) | 잘못 예측된 분기 버블 발생 빈도 | 2사이클/발생 |
| $rp$ (return penalty) | ret 버블 발생 빈도 | 3사이클/발생 |

**일반적인 프로그램에서의 추정**:
- 로드-사용 해저드: 명령어의 약 20%가 로드이고 그 중 약 25%가 즉시 사용 → lp ≈ 0.05
- 분기 명령어: 약 20%, 예측 실패율 약 40% → mp ≈ 0.16
- ret: 비교적 드묾 → rp ≈ 0.02
- **예상 CPI ≈ 1.0 + 0.05 + 0.16 + 0.02 = 1.23**

### 4.5.9 미완성 과제 (Unfinished Business)

PIPE는 실제 프로세서에 필요한 여러 기능이 없다:
- **멀티사이클 명령어**: 정수 곱셈/나눗셈, 부동소수점 연산
- **인터럽트 처리**: 외부 I/O 이벤트에 의한 예외 제어
- **메모리 계층**: L1/L2 캐시와 파이프라인의 통합
- **슈퍼스칼라(Superscalar)**: 매 사이클 여러 명령어 동시 발행

---

## 4.6 요약

```
┌───────────────────────────────────────────────────────────────┐
│                Chapter 4 핵심 개념 정리                        │
├───────────────┬───────────────────────────────────────────────┤
│ Y86-64 ISA    │ x86-64 단순화, 15레지스터, OPq/jXX/cmovXX/    │
│               │ call/ret/push/pop/halt                         │
├───────────────┼───────────────────────────────────────────────┤
│ HCL           │ bool/word 신호, case 표현식, in 연산자          │
├───────────────┼───────────────────────────────────────────────┤
│ SEQ           │ 6단계(Fetch→Decode→Execute→Memory→WB→PC)       │
│               │ 클로킹: PC, CC, 레지스터 파일, 데이터 메모리    │
├───────────────┼───────────────────────────────────────────────┤
│ 파이프라인    │ 처리량↑ / 지연 약간↑, 레지스터 오버헤드 주의   │
│ 원리          │ 비균일 단계 → 느린 단계가 병목                  │
├───────────────┼───────────────────────────────────────────────┤
│ PIPE 해저드   │ 데이터: 포워딩 + 로드-인터록(1사이클 스톨)      │
│               │ 제어: ret(3버블), 분기 오예측(2버블)             │
├───────────────┼───────────────────────────────────────────────┤
│ CPI           │ 1.0 + lp + mp + rp ≈ 1.2~1.3 (현실적)         │
└───────────────┴───────────────────────────────────────────────┘
```

### 핵심 교훈

- **ISA와 마이크로아키텍처의 분리**: ISA는 순차 실행처럼 보이지만 실제 하드웨어는 파이프라인으로 동작. 이 추상화가 컴파일러·프로그래머를 하드웨어 세부 사항에서 분리시킨다.
- **포워딩이 스톨보다 효율적**: 가능하면 스톨 없이 파이프라인 레지스터 값을 직접 전달
- **클로킹이 순서를 보장한다**: 조합 논리 + 클록 에지 캡처 = 올바른 순차 의미론 유지
- **예측과 취소**: 현대 프로세서는 분기 예측기(branch predictor)를 정교하게 만들어 mp를 최소화한다
