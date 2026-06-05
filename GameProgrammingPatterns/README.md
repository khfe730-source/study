# 게임 프로그래밍 패턴

> Robert Nystrom 저  
> GoF 디자인 패턴을 게임 개발 관점에서 재해석하고, 게임 엔진에서 실제로 사용되는 패턴을 다룬다.

---

## 설계 패턴 재방문 (Design Patterns Revisited)

| 챕터 | 패턴 | 핵심 |
|------|------|------|
| 03 | [Command](https://github.com/khfe730-source/study/tree/main/게임프로그래밍패턴/docs/03_command.md) | 요청을 객체로 캡슐화, undo/redo |
| 04 | [Flyweight](https://github.com/khfe730-source/study/tree/main/게임프로그래밍패턴/docs/04_flyweight.md) | 공유 상태로 메모리 절약 |
| 05 | [Observer](https://github.com/khfe730-source/study/tree/main/게임프로그래밍패턴/docs/05_observer.md) | 1:N 이벤트 통보 |
| 06 | [Prototype](https://github.com/khfe730-source/study/tree/main/게임프로그래밍패턴/docs/06_prototype.md) | 객체 복제, 데이터 프로토타입 |
| 07 | [Singleton](https://github.com/khfe730-source/study/tree/main/게임프로그래밍패턴/docs/07_singleton.md) | 안티패턴으로서의 싱글턴 |
| 08 | [State](https://github.com/khfe730-source/study/tree/main/게임프로그래밍패턴/docs/08_state.md) | FSM → HSM → Pushdown Automaton |

## 시퀀싱 패턴 (Sequencing Patterns)

| 챕터 | 패턴 | 핵심 |
|------|------|------|
| 09 | [Double Buffer](https://github.com/khfe730-source/study/tree/main/게임프로그래밍패턴/docs/09_double_buffer.md) | 불완전한 상태 노출 방지 |
| 10 | [Game Loop](https://github.com/khfe730-source/study/tree/main/게임프로그래밍패턴/docs/10_game_loop.md) | 프레임레이트 독립 루프 |
| 11 | [Update Method](https://github.com/khfe730-source/study/tree/main/게임프로그래밍패턴/docs/11_update_method.md) | 엔티티별 프레임 슬라이싱 |

## 행동 패턴 (Behavioral Patterns)

| 챕터 | 패턴 | 핵심 |
|------|------|------|
| 12 | [Bytecode](https://github.com/khfe730-source/study/tree/main/게임프로그래밍패턴/docs/12_bytecode.md) | 스택 머신 VM, 스크립트 샌드박스 |
| 13 | [Subclass Sandbox](https://github.com/khfe730-source/study/tree/main/게임프로그래밍패턴/docs/13_subclass_sandbox.md) | 기반 클래스가 연산 제공 |
| 14 | [Type Object](https://github.com/khfe730-source/study/tree/main/게임프로그래밍패턴/docs/14_type_object.md) | 데이터로 타입 정의 |

## 디커플링 패턴 (Decoupling Patterns)

| 챕터 | 패턴 | 핵심 |
|------|------|------|
| 15 | [Component](https://github.com/khfe730-source/study/tree/main/게임프로그래밍패턴/docs/15_component.md) | 도메인 분리, ECS의 토대 |
| 16 | [Event Queue](https://github.com/khfe730-source/study/tree/main/게임프로그래밍패턴/docs/16_event_queue.md) | 발신/수신 시간 디커플링, 링 버퍼 |
| 17 | [Service Locator](https://github.com/khfe730-source/study/tree/main/게임프로그래밍패턴/docs/17_service_locator.md) | 서비스 전역 접근, Null/Decorator |

## 최적화 패턴 (Optimization Patterns)

| 챕터 | 패턴 | 핵심 |
|------|------|------|
| 18 | [Data Locality](https://github.com/khfe730-source/study/tree/main/게임프로그래밍패턴/docs/18_data_locality.md) | 캐시 친화적 데이터 배치, 50배 성능 |
| 19 | [Dirty Flag](https://github.com/khfe730-source/study/tree/main/게임프로그래밍패턴/docs/19_dirty_flag.md) | 지연 계산으로 중복 연산 제거 |
| 20 | [Object Pool](https://github.com/khfe730-source/study/tree/main/게임프로그래밍패턴/docs/20_object_pool.md) | 재사용 풀, 파편화 방지, 프리 리스트 |
| 21 | [Spatial Partition](https://github.com/khfe730-source/study/tree/main/게임프로그래밍패턴/docs/21_spatial_partition.md) | 위치 기반 쿼리 최적화, Grid/Quadtree |

---

[← 전체 목록으로](https://github.com/khfe730-source/study/tree/main/README.md) &nbsp;|&nbsp; [전체 요약 보기](https://github.com/khfe730-source/study/tree/main/게임프로그래밍패턴/docs/00_summary.md)
