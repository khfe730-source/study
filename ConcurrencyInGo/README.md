# Concurrency in Go

**저자:** Katherine Cox-Buday  
**출판:** O'Reilly Media

Go의 동시성 모델(CSP 기반), 고루틴·채널·sync 패키지 등 빌딩 블록, 실전 패턴, 대규모 시스템에서의 동시성, Go 런타임 내부까지 다룬다.

---

## 챕터 목록

| 챕터 | 제목 | 핵심 내용 | 링크 |
|------|------|-----------|------|
| 1 | An Introduction to Concurrency | 레이스 컨디션, 원자성, 교착/라이브락/기아, 동시성 안전성 설계 | [바로가기](./docs/chapter01_AnIntroductionToConcurrency.md) |
| 2 | Modeling Your Code: Communicating Sequential Processes | 동시성 vs 병렬성, CSP, Go의 동시성 철학 | [바로가기](./docs/chapter02_ModelingYourCode_CSP.md) |
| 3 | Go's Concurrency Building Blocks | 고루틴, sync 패키지, 채널, select, GOMAXPROCS | [바로가기](./docs/chapter03_GoConcurrencyBuildingBlocks.md) |
| 4 | Concurrency Patterns in Go | Confinement, Pipeline, Fan-Out/In, or-done, tee, bridge, context | [바로가기](./docs/chapter04_ConcurrencyPatterns.md) |
| 5 | Concurrency at Scale | 에러 전파, 타임아웃/취소, 하트비트, 레이트 리미팅, 고루틴 복구 | - |
| 6 | Goroutines and the Go Runtime | Work Stealing, 런타임 스케줄러 내부 | - |
| A | Appendix | 고루틴 에러 해부, 레이스 감지, pprof | - |

---

## 핵심 개념 요약

### 동시성 문제의 분류

```
동시성 버그
├── 정확성(Correctness) 문제
│   ├── Race Condition     — 실행 순서 비보장
│   ├── Atomicity          — 연산의 불가분성 위반
│   └── Memory Sync        — 크리티컬 섹션 미보호
└── 진행성(Progress) 문제
    ├── Deadlock           — 모두 서로를 대기 (코프만 4조건)
    ├── Livelock           — 활발하지만 진전 없음
    └── Starvation         — 일부가 자원을 독점
```

### Go의 동시성 철학

> *"Do not communicate by sharing memory; instead, share memory by communicating."*

채널(channel)을 통한 통신을 우선하되, 공유 메모리가 더 명확한 경우 sync 패키지를 사용한다.
