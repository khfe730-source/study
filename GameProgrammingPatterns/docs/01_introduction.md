# 서론 (Introduction)

> **Game Programming Patterns** — Robert Nystrom 저  
> Part I: Introduction

---

## 저자의 이야기

저자 Bob Nystrom은 초등학교 5학년 때 낡은 **TRS-80** 컴퓨터를 접하며 처음으로 프로그래밍을 경험했다. 카세트 드라이브가 고장난 탓에 코드를 매번 처음부터 손으로 입력해야 했고, 덕분에 짧은 프로그램을 선호하게 되었다.

```basic
10 PRINT "BOBBY IS RADICAL!!!"
20 GOTO 10
```

페이지 더미 맨 뒤에는 수 페이지에 달하는 "Tunnels and Trolls"라는 프로그램이 있었다. 실행에는 끝내 실패했지만, 그 순간부터 그는 **게임 프로그래머**를 꿈꾸게 되었다.

십대 시절 Macintosh와 QuickBASIC, THINK C를 접한 후 여름방학마다 게임을 만들었다. 프로그램이 커질수록 구조화의 필요성을 느꼈고, "C++ 사용법"이 아니라 **"프로그램을 어떻게 구성하는가"**에 관한 책을 찾기 시작했다.

그러다 **Gang of Four**의 명저 『Design Patterns: Elements of Reusable Object-Oriented Software』를 한 번에 정독하며 비로소 돌파구를 찾았다.

2001년 Electronic Arts에 입사한 뒤, 실제 상용 게임 코드를 열어보며 깨달음과 실망을 동시에 경험했다:

> 그래픽, AI, 애니메이션 분야의 코드는 눈부시게 훌륭했다.  
> 하지만 그 코드들을 지탱하는 **아키텍처(architecture)** 는 종종 뒷전이었다.  
> 모듈 간 커플링(coupling)은 만연했고, 새 기능은 어디든 끼워 넣기 식이었다.

---

## 이 책의 목적

이 책은 좋은 코드에 묻혀 있던 패턴들을 발굴해 정리한 것이다.

- **도메인 특화 서적**(그래픽, AI, 물리 등)도 아니고
- **엔진 전체를 다루는 서적**도 아니다.

**각 챕터가 독립적인 아이디어**로 구성되어 있어, 자신의 프로젝트에 필요한 것만 골라 적용하는 **à la carte** 방식이다.

---

## Design Patterns와의 관계

이 책은 Gang of Four의 『Design Patterns』과 명백한 관계를 가진다.

- **Design Patterns Revisited** 섹션에서는 GoF의 여러 패턴을 게임 프로그래밍 관점으로 재조명한다.
- 이 책에서 새롭게 소개하는 패턴들은 게임 특유의 엔지니어링 문제에 특히 잘 맞는다:

| 게임의 특성 | 설명 |
|---|---|
| 시간과 순서 (Time & Sequencing) | 정해진 순서와 타이밍이 아키텍처의 핵심이다 |
| 압축된 개발 사이클 | 여러 프로그래머가 빠르게 행동을 정의하고 이터레이션해야 한다 |
| 복잡한 상호작용 | 몬스터, 아이템, 폭발 등 다양한 시스템이 서로 엮인다 |
| 성능 | 플랫폼에서 최대한의 성능을 끌어내는 것이 게임의 생명이다 |

---

## 책 구성

```
Part I   - Introduction (서론)
           Ch1. Architecture, Performance, and Games

Part II  - Design Patterns Revisited (GoF 패턴 재조명)
           Ch2. Command
           Ch3. Flyweight
           Ch4. Observer
           Ch5. Prototype
           Ch6. Singleton
           Ch7. State

Part III - Sequencing Patterns (순서 패턴)
           Ch8.  Double Buffer
           Ch9.  Game Loop
           Ch10. Update Method

Part IV  - Behavioral Patterns (행동 패턴)
           Ch11. Bytecode
           Ch12. Subclass Sandbox
           Ch13. Type Object

Part V   - Decoupling Patterns (디커플링 패턴)
           Ch14. Component
           Ch15. Event Queue
           Ch16. Service Locator

Part VI  - Optimization Patterns (최적화 패턴)
           Ch17. Data Locality
           Ch18. Dirty Flag
           Ch19. Object Pool
           Ch20. Spatial Partition
```

---

## 각 챕터의 구성 방식

모든 패턴 챕터는 다음의 일관된 구조로 작성되어 있다:

| 섹션 | 설명 |
|---|---|
| **Intent (의도)** | 패턴이 해결하려는 문제를 한 문장으로 요약 |
| **Motivation (동기)** | 패턴을 적용할 구체적인 예제 문제 제시 |
| **The Pattern (패턴)** | 패턴의 본질을 건식(dry) 텍스트 형태로 설명 |
| **When to Use It (사용 시기)** | 패턴이 유용한 상황과 피해야 할 상황 |
| **Keep in Mind (주의사항)** | 패턴 사용 시 결과와 리스크 |
| **Sample Code (예제 코드)** | 단계별 전체 구현 예제 (C++) |
| **Design Decisions (설계 결정)** | 패턴 적용 시 고려할 다양한 옵션 |
| **See Also (참고)** | 관련 패턴 및 실제 오픈소스 사례 |

---

## 예제 코드에 대하여

- 모든 코드 예제는 **C++** 로 작성되어 있다.
- 이유: 상용 게임에서 가장 널리 쓰이는 언어이며, Java, C#, JavaScript 등과 문법 유사성이 높다.
- **모던 C++ (C++11 이상) 스타일을 의도적으로 피한다.** 표준 라이브러리와 템플릿을 거의 사용하지 않아, 다른 언어 사용자도 이해하기 쉽게 구성되었다.
- 코드의 목적은 언어 기법이 아닌 **아이디어 표현**임을 명심할 것.

---

*© 2009-2015 Robert Nystrom*
