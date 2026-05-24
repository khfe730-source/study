# Chapter 16: Security

## 보안(Security) vs 보호(Protection)

- **보안(Security)**: 시스템과 데이터의 무결성이 유지된다는 신뢰의 척도. 외부 위협까지 포함
- **보호(Protection)**: 프로세스·사용자의 자원 접근을 제어하는 메커니즘의 집합 (Ch17)

---

## 16.1 보안 문제 (The Security Problem)

### 보안 침해 유형

| 침해 유형 | 설명 |
|----------|------|
| **기밀성 침해(Breach of Confidentiality)** | 데이터 무단 읽기·탈취 (신용카드 정보, ID 도용) |
| **무결성 침해(Breach of Integrity)** | 데이터 무단 수정 (소스 코드 변조, 책임 전가) |
| **가용성 침해(Breach of Availability)** | 데이터 무단 파괴 (웹 사이트 변조) |
| **서비스 도용(Theft of Service)** | 자원의 무단 사용 (악성 파일 서버 데몬 설치) |
| **서비스 거부(Denial of Service)** | 합법적 시스템 사용 방해 (DoS/DDoS) |

### 주요 공격 방법

```
정상 통신:
  Sender ←──────────────────── Receiver

위장 공격 (Masquerading):
  Attacker [←→ Sender로 위장] ──→ Receiver

중간자 공격 (Man-in-the-Middle):
  Sender ──→ [Attacker] ──→ Receiver
              (도청·변조)

재전송 공격 (Replay Attack):
  캡처된 유효 데이터를 나중에 재전송

세션 하이재킹 (Session Hijacking):
  진행 중인 활성 세션 가로채기 → MITM의 전제 조건

권한 상승 (Privilege Escalation):
  이메일 스크립트 실행으로 발신자 권한 초과
```

### 4계층 보안 모델

```
┌─────────────────────────────────────────┐
│  4. Application Layer                   │
│  (3rd-party 앱 취약점, 설정 오류)        │
├─────────────────────────────────────────┤
│  3. Operating System Layer              │
│  (패치, 하드닝, 공격 표면 최소화)        │
├─────────────────────────────────────────┤
│  2. Network Layer                       │
│  (네트워크 도청·차단, IoT 기기)          │
├─────────────────────────────────────────┤
│  1. Physical Layer                      │
│  (서버실 물리적 출입 통제)               │
└─────────────────────────────────────────┘
```

> 보안은 가장 약한 고리만큼만 강하다. 인간 요소(social engineering, phishing)도 필수 고려.

---

## 16.2 프로그램 위협 (Program Threats)

### 16.2.1 악성 소프트웨어 (Malware)

| 유형 | 설명 |
|------|------|
| **트로이 목마(Trojan Horse)** | 정상 기능처럼 보이지만 악성 행위 수행 (플래시라이트 앱이 연락처 탈취) |
| **스파이웨어(Spyware)** | 사용자 정보 수집·팝업·광고 삽입; 스팸 릴레이 |
| **랜섬웨어(Ransomware)** | 파일 암호화 후 복호화 키 대가로 몸값 요구 |
| **트랩 도어(Trap Door / Back Door)** | 개발자가 숨겨놓은 비밀 접근 경로 |
| **논리 폭탄(Logic Bomb)** | 특정 조건 충족 시 활성화되는 트랩 도어 |

**최소 권한 원칙(Principle of Least Privilege):**
> "모든 프로그램과 권한 있는 사용자는 작업 완료에 필요한 최소한의 권한만 사용해야 한다." — Saltzer (Multics, 1974)

- 악성 코드가 root 권한을 못 얻으면 피해 범위 제한 가능
- 코드 리뷰(code review)로 트랩 도어 탐지 가능 (git 등 버전 관리 도구 활용)

### 16.2.2 코드 인젝션 (Code Injection)

버퍼 오버플로우(buffer overflow)가 대표적 코드 인젝션 벡터:

```c
// 취약한 코드 (BUFFER_SIZE=0으로 정의됨)
char buffer[BUFFER_SIZE];
strcpy(buffer, argv[1]);  // 경계 검사 없음 → 오버플로우

// 안전한 코드
strncpy(buffer, argv[1], sizeof(buffer) - 1);
```

**오버플로우 결과:**

```
스택 레이아웃:
  [buffer 0..N][padding][other vars][... stack frame ...][return address]

오버플로우 크기에 따른 결과:
  ① 소량 → padding 범위 내 → 무해
  ② 중간 → 인접 변수 덮어씀 → 예상치 못한 동작·크래시
  ③ 대량 → return address 덮어씀 → 공격자가 임의 코드 실행 가능

공격 흐름 (shellcode 인젝션):
  1. 공격자: execvp("/bin/sh") 호출 shellcode 작성
  2. NOP-sled 삽입: [NOP...NOP][shellcode]
  3. return address → NOP-sled 시작 주소로 덮어씀
  4. 함수 반환 시 NOP-sled → shellcode 실행
  5. 공격 프로세스 권한으로 shell 획득
```

**방어 기법:**
- `strncpy()`, `snprintf()` 등 크기 제한 함수 사용
- **ASLR(Address Space Layout Randomization)**: 스택/힙 시작 주소를 무작위화 → 주소 예측 불가

힙 오버플로우, use-after-free, double-free도 동일한 코드 인젝션 벡터로 악용 가능.

### 16.2.3 바이러스와 웜 (Viruses and Worms)

| 구분 | 설명 |
|------|------|
| **바이러스(Virus)** | 합법적 프로그램에 삽입된 자가 복제 코드; 인간 활동 필요 |
| **웜(Worm)** | 네트워크를 통해 인간 개입 없이 자가 복제 |

**바이러스 분류:**

```
파일 바이러스(File):     프로그램 파일 앞에 삽입, 제어 탈취 후 원본 실행
부트 바이러스(Boot):     부트 섹터 감염, 부팅마다 실행, 파일 시스템 밖에 존재
매크로 바이러스(Macro):  Word/Excel 매크로로 구현, 높은 수준 언어 사용
루트킷(Rootkit):         OS 자체 침투, 탐지 기능 포함한 모든 시스템 기능 장악
소스 코드(Source Code):  소스 파일 수정하여 바이러스 삽입
다형성(Polymorphic):     설치마다 코드 변형 → 시그니처 탐지 회피
암호화(Encrypted):       복호화 코드 + 암호화 바이러스 본체
은폐(Stealth):           read() 시스템 콜 수정, 감염 사실 숨김
다중 감염(Multipartite): 부트 섹터·메모리·파일 동시 감염
방어형(Armored):         역분석 방해, 압축으로 탐지 회피
```

---

## 16.3 시스템 및 네트워크 위협 (System and Network Threats)

### 좀비 시스템 (Zombie Systems)
- 해커에 의해 침해된 제3의 시스템; 소유자 모르게 DDoS·스팸 릴레이에 악용
- 공격 출처 추적 어렵게 만드는 역할 → "사소한" 시스템도 보안 중요

### 16.3.1 네트워크 트래픽 공격

```
수동적 공격 (Passive):
  스니핑(Sniffing): 네트워크 트래픽 도청

능동적 공격 (Active):
  스푸핑(Spoofing):  한쪽 당사자로 위장
  MITM: 발신자→공격자→수신자 (양쪽 모두 위장)
```

### 16.3.2 서비스 거부 (Denial of Service)

- **DoS**: 단일 소스에서 대상 시스템 과부하
- **DDoS(Distributed DoS)**: 좀비 네트워크(botnet)에서 동시 공격 → 방어 어려움
- 방화벽 자동 차단 규칙도 악용 가능 → 정상 트래픽도 차단되는 부작용

### 16.3.3 포트 스캐닝 (Port Scanning)

- 공격 자체는 아니나 취약점 정찰(reconnaissance) 수단
- **핑거프린팅(Fingerprinting)**: OS 종류·서비스 버전 추론 → 알려진 취약점 매핑
- 도구: `nmap` (오픈소스), Metasploit (취약점 익스플로잇)
- 이상 탐지(anomaly detection)로 포트 스캔 탐지 가능

---

## 16.4 암호학 (Cryptography as a Security Tool)

암호학의 목적: 신뢰할 수 없는 네트워크에서 메시지 수신자/발신자를 제한

### 16.4.1 암호화 (Encryption)

암호화 알고리즘의 구성: 키 집합 K, 메시지 집합 M, 암호문 집합 C, 암호화 함수 E, 복호화 함수 D

#### 대칭 암호화 (Symmetric Encryption)

```
동일한 키 k로 암호화·복호화:

  Sender                              Receiver
  E_k(m) = c ──── insecure channel ──→ D_k(c) = m
                      (c 노출)

키 교환이 문제: 안전하게 키를 사전 공유해야 함
```

| 알고리즘 | 키 길이 | 블록 크기 | 상태 |
|----------|:-------:|:--------:|------|
| DES | 56-bit | 64-bit | 취약 (완전 탐색 가능) |
| 3DES | 112/168-bit | 64-bit | 보통 (여전히 사용) |
| **AES** (Rijndael) | 128/192/256-bit | 128-bit | **현재 표준** (FIPS-197) |

**스트림 암호(Stream Cipher)**: 바이트/비트 스트림 암호화; keystream과 XOR

#### 비대칭 암호화 (Asymmetric Encryption)

```
공개키(Public Key, ke):  누구나 가질 수 있음 (암호화용)
개인키(Private Key, kd): 수신자만 보유 (복호화용)

RSA 암호화:
  암호화: c = m^ke mod N
  복호화: m = c^kd mod N
  N = p × q (p, q: 큰 소수)
  ke × kd ≡ 1 (mod (p-1)(q-1))

예시 (p=7, q=13, N=91):
  ke=5, kd=29
  메시지 69 암호화: 69^5 mod 91 = 62
  복호화: 62^29 mod 91 = 69
```

- 비대칭 암호화는 계산 비용이 높아 대량 데이터 직접 암호화에는 비적합
- 주로 **소량 데이터 암호화, 인증, 키 교환**에 사용

#### 인증 (Authentication)

```
발신자를 제한하는 메커니즘

해시 함수 H(m):
  • 고정 크기의 메시지 다이제스트(message digest) 생성
  • 충돌 저항(collision-resistant): H(m) = H(m') → m = m' (역)
  • MD5: 128-bit (취약), SHA-1: 160-bit

MAC (Message Authentication Code):
  • 대칭키를 사용한 암호학적 체크섬
  • H(m)을 인증하면 긴 메시지도 안전하게 인증 가능

디지털 서명 (Digital Signature):
  • 개인키로 서명 생성: Sig = H(m)^ks mod N
  • 공개키로 검증: V_kv(m, a) = (a^kv mod N == H(m))
  • 누구나 검증 가능 (공개키 사용)
  • 부인 방지(Non-repudiation) 구현의 핵심
```

#### 키 배포 (Key Distribution)

```
중간자 공격 위험:
  수신자가 공개키 배포 → 공격자가 가짜 공개키 삽입
  → 발신자가 공격자 키로 암호화 → 공격자 복호화 가능

해결책: 디지털 인증서 (Digital Certificate, X.509)
  • 신뢰할 수 있는 인증 기관(CA)이 공개키에 디지털 서명
  • 웹 브라우저에 CA 공개키 미리 포함 (신뢰 앵커)
  • 신뢰 체계(web of trust): CA가 다른 CA를 인증
```

### 16.4.2 암호화 구현 레이어

| 레이어 | 프로토콜 | 특징 |
|--------|---------|------|
| 응용 계층 | PGP (이메일), S/MIME | 응용별 구현 |
| 전송 계층 | **TLS/SSL** | 현재 가장 광범위 사용 |
| 네트워크 계층 | **IPSec** | 패킷 레벨 암호화, VPN 기반 |

### 16.4.3 TLS 프로토콜 예시

```
TLS 핸드셰이크 흐름:
  Client (C)                              Server (S)
     │                                        │
     │── nc (28-byte random) ────────────────→│
     │                                        │
     │←── ns (random) + cert_s (서버 인증서) ─│
     │                                        │
     │ cert 검증: CA 서명 확인 + 유효 기간 확인│
     │                                        │
     │ pms (46-byte premaster secret) 생성    │
     │── cpms = E_ke(pms) ─────────────────→ │
     │                      D_kd(cpms) = pms  │
     │                                        │
  양쪽 ms = H(nc, ns, pms) 계산 (공유 마스터 시크릿)
     │                                        │
  세션 키 도출:
    k_crypt_cs, k_crypt_sc (암호화 키)
    k_mac_cs,   k_mac_sc   (MAC 키)
     │                                        │
  이후 통신: E_k_crypt(m || S_k_mac(m)) 형태
```

- 비대칭 암호화로 키 교환 → 대칭 암호화로 세션 데이터 처리
- 세션 종료 후 세션 키 폐기 (Forward Secrecy)
- TLS VPN: 유연하지만 IPSec VPN보다 효율 낮음

---

## 16.5 사용자 인증 (User Authentication)

### 16.5.1~16.5.2 패스워드와 취약점

**패스워드 공격 방법:**

| 방법 | 설명 |
|------|------|
| **추측(Guessing)** | 사용자 정보 기반 (이름, 생일 등) |
| **무차별 대입(Brute Force)** | 4자리 숫자: ~10,000가지, 1ms/시도 → 5초 |
| **사전 공격(Dictionary Attack)** | 단어·변형·일반 패스워드 목록 시도 |
| **숄더 서핑(Shoulder Surfing)** | 어깨 너머로 키보드 관찰 |
| **스니핑(Sniffing)** | 네트워크 패킷에서 패스워드 캡처 |
| **스키머(Skimmer)** | ATM 등에 물리적 기기 설치 |

### 16.5.3 패스워드 보호 (Securing Passwords)

UNIX 해시 방식:

```
저장: hash(password + salt) → 패스워드 파일에 저장
인증: 입력 패스워드 + 동일 salt → hash → 저장값과 비교

salt의 역할:
  • 동일 패스워드가 다른 해시값 → 사전 공격 무력화
  • 사전 단어를 모든 salt와 조합해야 하므로 레인보우 테이블 무효화
  • UNIX: /etc/shadow (root만 읽기 가능), setuid 프로그램으로 비교
```

### 16.5.4 일회용 패스워드 (One-Time Passwords)

```
챌린지-응답 방식:
  System: ch (challenge) 제시
  User:   H(pw, ch) 계산 → 응답
  Next:   새로운 ch → 응답도 달라짐

특징:
  • 패킷 캡처로 재사용 불가 (replay attack 방어)
  • 하드웨어 토큰(OTP 생성기), 스마트폰 앱으로 구현
  • 2단계 인증(2FA): PIN + OTP 생성기 (소유물 + 지식)
```

### 16.5.5 생체 인증 (Biometrics)

- 지문 인식: 융선(ridge) 패턴 → 숫자 시퀀스 변환, 비교
- 2FA + 생체: USB 장치 + PIN + 지문 → 강력한 다중 인증
- **주의**: 강력한 인증도 세션 암호화 없이는 세션 하이재킹에 취약

---

## 16.6 보안 방어 구현 (Implementing Security Defenses)

### 16.6.1 보안 정책 (Security Policy)

- 무엇을 보호할지 명시하는 문서
- 예: "외부 접근 가능 앱은 배포 전 코드 리뷰 필수", "비밀번호 공유 금지"
- 정기적으로 검토·업데이트 필요 (생동하는 문서)

### 16.6.2 취약점 평가 (Vulnerability Assessment)

**침투 테스트(Penetration Test)**: 알려진 취약점 스캔

시스템 내부 스캔 항목:
- 짧거나 쉽게 추측 가능한 패스워드
- 승인되지 않은 특권 프로그램(setuid 등)
- 예상치 못하게 장기 실행 중인 프로세스
- 시스템 파일(패스워드 파일, 커널 등) 부적절한 보호
- 프로그램 탐색 경로에 위험한 항목(`/tmp`, 현재 디렉토리)
- 체크섬으로 감지된 시스템 프로그램 변경

### 16.6.3 침입 방지 (Intrusion Prevention)

**시그니처 기반 탐지(Signature-Based Detection):**
- 알려진 공격 패턴 스캔 (예: 네트워크 패킷에서 `/etc/passwd` 탐색)
- 새로운 제로데이 공격 탐지 불가

**이상 탐지(Anomaly Detection):**
- 정상 행동 벤치마크 → 이탈 행동 탐지
- 제로데이 공격 탐지 가능
- 과거 침해 시 오염된 벤치마크 문제

**Bayes 정리 기반 오경보율 분석:**

```
P(I) = 0.00002 (실제 침해 레코드 비율)
P(A|I) = 0.8  (실제 침해 탐지율)
P(A|¬I) = 0.0001 (오경보율)

P(I|A) = P(I)·P(A|I) / (P(I)·P(A|I) + P(¬I)·P(A|¬I))
        ≈ 0.14 (경보 7번 중 실제 침해 1번)

→ IPS는 극히 낮은 오경보율이 필수!
```

### 16.6.4 바이러스 방어

- 알려진 패턴 스캔 → 패턴 계열(families)로 확장
- **샌드박스(Sandbox)**: 제어된 환경에서 코드 실행 후 행동 분석
- 바이러스 서명(signature) 정기 업데이트 필수
- 예방이 최선: 신뢰 출처에서만 소프트웨어 설치, 첨부 파일 주의
- 메시지 다이제스트(checksum) 비교로 변경 탐지

### 16.6.5 감사·기록 (Auditing, Accounting, Logging)

- 인증/권한 부여 실패 로그 → 침입 시도 탐지
- 과금(accounting) 로그의 이상 → 보안 문제 조기 발견 (Cliff Stoll의 사례)

### 16.6.6 방화벽 (Firewalling)

```
DMZ(비무장 지대) 아키텍처:

  Internet ──→ [Firewall 1] ──→ DMZ ──→ [Firewall 2] ──→ Company Network
                                 (웹 서버,                   (내부 DB 등)
                                  메일 서버)

  Internet → DMZ: 허용
  Company → Internet: 허용
  Internet → Company: 차단
  DMZ → Company: 제한적 허용 (예: 웹서버 → DB)
```

| 방화벽 종류 | 설명 |
|------------|------|
| **네트워크 방화벽** | 도메인 간 통신 제한; 출발지·목적지 IP/포트 기반 |
| **개인 방화벽** | 호스트 단위; OS 포함 또는 별도 앱 |
| **애플리케이션 프록시** | 특정 프로토콜(SMTP 등)을 이해하여 심층 검사 |
| **XML 방화벽** | XML 트래픽 전용 분석 |
| **시스템 콜 방화벽** | 앱과 커널 사이; 시스템 콜 모니터링 (Solaris "least privilege") |

**방화벽 한계**: 허용된 프로토콜 내 공격(예: HTTP를 통한 버퍼 오버플로우)은 차단 불가

### 16.6.7 기타 방어 기법

- **ASLR(Address Space Layout Randomization)**: 스택·힙 주소 무작위화 → 코드 인젝션 어렵게
  - Windows, Linux, macOS 표준 기능
- **시스템 파티션 분리**: iOS/Android에서 시스템(읽기 전용)과 데이터(읽기-쓰기) 파티션 분리
  - Android의 dm-verity: 시스템 파티션 암호화 해시로 변조 탐지

### 16.6.8 보안 방어 요약

- 사용자 교육 (강력한 비밀번호, 피싱 방어)
- OS 공격 표면 최소화 (불필요한 서비스 비활성화)
- 시스템·앱 최신 패치 유지
- 신뢰할 수 있는 출처(코드 서명)의 앱만 실행
- 로그 활성화·주기적 검토
- 안티바이러스 설치·업데이트
- 방화벽, 침입 탐지 시스템 운용
- 중요 데이터 암호화 (전체 디스크, 개별 파일)

---

## 16.7 예시: Windows 10 보안

| 개념 | 설명 |
|------|------|
| **보안 ID(Security ID)** | 각 사용자의 고유 식별자 |
| **보안 접근 토큰(Access Token)** | 로그온 시 생성; 사용자·그룹 SID + 특수 권한 목록 |
| **주체(Subject)** | 사용자 접근 토큰 + 프로그램 조합; 단순/서버 주체 구분 |
| **보안 설명자(Security Descriptor)** | 소유자 SID + 그룹 SID + DACL + SACL + 무결성 레이블 |
| **DACL(임의 접근 제어 목록)** | 허용/거부 항목; ReadData/WriteData/Execute 등 세부 제어 |
| **SACL(시스템 접근 제어 목록)** | 감사 메시지 생성 제어; 무결성 레이블 설정 |

**Windows Vista+ 무결성 레이블(Mandatory Integrity Control):**

```
레이블 순서: untrusted → low → medium → high → system

규칙: NoWriteUp 자동 적용 → 낮은 무결성 주체는 높은 객체에 쓰기 불가

UAC(User Account Control):
  관리자 계정: 두 개의 토큰
  ① 보통 사용: Administrators 그룹 비활성화 + medium 레이블
  ② 상승 사용: Administrators 그룹 활성화 + high 레이블
```

**코드 서명(Code Signing):** Windows 10 일부 버전에서 필수화 — 미서명 앱 실행 불가

---

## 핵심 개념 정리

### 암호화 방식 비교

| 구분 | 대칭 | 비대칭 |
|------|:----:|:-----:|
| 키 개수 | 1 (공유) | 2 (공개/개인) |
| 속도 | 빠름 | 느림 |
| 키 배포 | 어려움 | 쉬움 |
| 주요 용도 | 대량 데이터 암호화 | 키 교환, 인증, 서명 |
| 대표 알고리즘 | AES, 3DES | RSA, ECC |

### 인증 방법 비교

| 방법 | 요소 | 대표 약점 |
|------|------|---------|
| 패스워드 | 지식 | 추측, 스니핑, 재사용 |
| 일회용 패스워드 | 지식 + 소유물 | 기기 분실 |
| 2FA | 지식 + 소유물 | 세션 하이재킹 |
| 생체 인식 | 속성 | 위조 어려움, 갱신 불가 |
| 다중 인증 (MFA) | 복합 | 가장 강력한 방어 |

### 공격-방어 대응 관계

```
버퍼 오버플로우 → ASLR + 경계 검사 함수 + 스택 보호
바이러스      → 안티바이러스 + 샌드박스 + 코드 서명
DoS/DDoS      → 업스트림 필터링 + 트래픽 제한 (완전 방어 불가)
네트워크 도청  → TLS/IPSec 암호화
MITM          → 디지털 인증서 (CA 서명)
패스워드 탈취  → salt + hash + 2FA + 일회용 패스워드
```
