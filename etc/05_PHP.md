# PHP 핵심 정리 (서버 개발 중심)

---

## 1. PHP 기본 특징

- 동적 타입 언어, 인터프리터 방식 (OPcache로 바이트코드 캐싱)
- 웹 요청마다 스크립트 실행 후 메모리 해제 (Shared-Nothing)
- PHP 7+: JIT 컴파일러, 타입 선언 강화, 성능 대폭 향상
- PHP 8+: Named Arguments, Fibers, Union Types, Match 표현식

---

## 2. 타입 시스템

```php
// PHP 8 타입 선언
function add(int $a, int $b): int {
    return $a + $b;
}

// Union 타입 (PHP 8.0)
function process(int|string $input): int|string { ... }

// Nullable
function find(?int $id): ?User { ... }

// 타입 강제 비교 주의
var_dump(0 == "foo");  // PHP 7: true (느슨한 비교), PHP 8: false
var_dump(0 === "foo"); // false (엄격한 비교)

// 타입 캐스팅
$int = (int)"42abc";     // 42
$float = (float)"3.14";  // 3.14
```

---

## 3. OOP

```php
// 추상 클래스와 인터페이스
interface Playable {
    public function play(): void;
}

abstract class Character {
    abstract public function attack(): int;
    
    public function die(): void {
        echo "Character died";
    }
}

// 트레이트 (Trait) - 다중 상속 대안
trait Serializable {
    public function toJson(): string {
        return json_encode(get_object_vars($this));
    }
}

class Player extends Character implements Playable {
    use Serializable;
    
    public function __construct(
        private string $name,  // PHP 8 생성자 프로퍼티 프로모션
        private int $hp = 100
    ) {}
    
    public function attack(): int { return 10; }
    public function play(): void { echo "playing"; }
}
```

---

## 4. 배열과 컬렉션

```php
// PHP 배열은 OrderedMap
$arr = ['a' => 1, 'b' => 2, 'c' => 3];

// 유용한 배열 함수
array_map(fn($x) => $x * 2, $arr);
array_filter($arr, fn($x) => $x > 1);
array_reduce($arr, fn($carry, $x) => $carry + $x, 0);
usort($arr, fn($a, $b) => $a - $b);

// Spread 연산자
$merged = [...$arr1, ...$arr2];

// array_chunk, array_slice, array_splice
$chunks = array_chunk($arr, 3);
$slice = array_slice($arr, 1, 2);
```

---

## 5. 에러 처리

```php
// 예외 처리
try {
    if ($divide === 0) {
        throw new InvalidArgumentException("Division by zero");
    }
} catch (InvalidArgumentException $e) {
    echo $e->getMessage();
} catch (Exception $e) {
    echo "General error: " . $e->getMessage();
} finally {
    // 항상 실행
}

// 커스텀 예외
class GameException extends RuntimeException {
    public function __construct(
        private int $errorCode,
        string $message = ""
    ) {
        parent::__construct($message);
    }
    
    public function getErrorCode(): int {
        return $this->errorCode;
    }
}

// set_error_handler로 에러를 예외로 변환
set_error_handler(function($errno, $errstr) {
    throw new ErrorException($errstr, $errno);
});
```

---

## 6. 데이터베이스 연동 (PDO)

```php
// PDO 연결
$pdo = new PDO(
    "mysql:host=localhost;dbname=game;charset=utf8mb4",
    $user, $password,
    [PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
     PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC]
);

// Prepared Statement (SQL 인젝션 방지)
$stmt = $pdo->prepare("SELECT * FROM users WHERE id = :id AND status = :status");
$stmt->execute(['id' => $userId, 'status' => 1]);
$user = $stmt->fetch();

// 트랜잭션
try {
    $pdo->beginTransaction();
    $pdo->prepare("UPDATE users SET gold = gold - ? WHERE id = ?")->execute([100, $userId]);
    $pdo->prepare("INSERT INTO items (user_id, item_id) VALUES (?, ?)")->execute([$userId, $itemId]);
    $pdo->commit();
} catch (Exception $e) {
    $pdo->rollBack();
    throw $e;
}
```

---

## 7. 세션과 쿠키

```php
// 세션
session_start();
$_SESSION['user_id'] = 123;
$_SESSION['token'] = bin2hex(random_bytes(32));
session_destroy();

// 보안 설정
ini_set('session.cookie_httponly', 1);
ini_set('session.cookie_secure', 1);   // HTTPS에서만
ini_set('session.use_strict_mode', 1);

// 세션 하이재킹 방지
session_regenerate_id(true);  // 로그인 성공 후
```

---

## 8. 성능 최적화

**OPcache 설정 (php.ini)**
```ini
opcache.enable=1
opcache.memory_consumption=256
opcache.max_accelerated_files=20000
opcache.revalidate_freq=60
```

**메모리/성능 팁**
```php
// 참조로 큰 배열 전달
function process(array &$bigArray) { ... }

// unset으로 불필요한 변수 해제
unset($largeObject);

// Generator로 메모리 절약
function getLines(string $file): Generator {
    $f = fopen($file, 'r');
    while ($line = fgets($f)) {
        yield $line;
    }
    fclose($f);
}
foreach (getLines('huge.log') as $line) { ... }

// 스트링 연결 대신 배열+implode
$parts = [];
foreach ($items as $item) {
    $parts[] = $item->toString();
}
echo implode(', ', $parts);
```

---

## 9. Composer와 PSR

**Composer 명령**
```bash
composer install        # composer.lock 기준 설치
composer update         # 최신 버전으로 업데이트
composer require guzzlehttp/guzzle
composer dump-autoload  # 오토로더 재생성
```

**PSR 주요 표준**
- **PSR-1**: 기본 코딩 표준 (StudlyCaps, camelCase)
- **PSR-4**: 오토로딩 (네임스페이스 → 파일 경로 매핑)
- **PSR-7**: HTTP 메시지 인터페이스
- **PSR-12**: 코딩 스타일 가이드

---

## 10. 게임 서버에서 PHP 패턴

```php
// API 응답 표준화
class ApiResponse {
    public static function success(array $data, int $code = 200): array {
        return ['code' => 0, 'data' => $data, 'http_status' => $code];
    }
    
    public static function error(int $errorCode, string $msg): array {
        return ['code' => $errorCode, 'message' => $msg, 'http_status' => 400];
    }
}

// Rate Limiting (Redis 기반)
class RateLimiter {
    public function check(string $key, int $limit, int $window): bool {
        $current = $this->redis->incr($key);
        if ($current === 1) {
            $this->redis->expire($key, $window);
        }
        return $current <= $limit;
    }
}

// 분산 락 (Redis)
class DistributedLock {
    public function acquire(string $key, int $ttl): bool {
        return (bool) $this->redis->set(
            "lock:$key", 1,
            ['NX' => true, 'EX' => $ttl]
        );
    }
    
    public function release(string $key): void {
        $this->redis->del("lock:$key");
    }
}
```

---

## 11. 자주 나오는 면접 질문

**Q. PHP의 Shared-Nothing 아키텍처란?**
각 HTTP 요청이 독립적인 PHP 프로세스에서 처리되고, 요청 종료 후 메모리가 해제됨. 요청 간 상태를 공유하지 않아 확장성에 유리하지만, 공유 상태는 DB나 캐시(Redis)를 통해야 함.

**Q. PHP-FPM이란?**
FastCGI Process Manager. PHP 프로세스 풀을 관리해 웹서버(Nginx)와 통신. pm.max_children, pm.start_servers 등 프로세스 설정 조정으로 성능 튜닝.

**Q. SQL Injection 방어?**
Prepared Statement 사용. 입력값을 직접 쿼리에 삽입하지 않고 바인딩 파라미터 사용.

**Q. XSS 방어?**
출력 시 `htmlspecialchars()` 적용. Content-Security-Policy 헤더 설정.
