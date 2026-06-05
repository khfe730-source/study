# 자료구조 & 알고리즘 핵심 정리

---

## 1. 시간/공간 복잡도

| 표기 | 의미 | 예시 |
|------|------|------|
| O(1) | 상수 | 배열 인덱스 접근, 해시맵 조회 |
| O(log n) | 로그 | 이진 탐색, 균형 BST |
| O(n) | 선형 | 배열 순회 |
| O(n log n) | 선형 로그 | 병합 정렬, 퀵 정렬(평균) |
| O(n²) | 이차 | 버블/선택/삽입 정렬 |
| O(2ⁿ) | 지수 | 부분집합 열거, 완전탐색 |

---

## 2. 배열 & 슬라이스

```
장점: 인덱스 O(1) 접근, 캐시 친화적(연속 메모리)
단점: 삽입/삭제 O(n) (이동 필요), 크기 고정(정적 배열)

동적 배열(ArrayList/Slice): 용량 초과 시 2배 확장
→ 분할 상환 분석으로 append는 평균 O(1)
```

**투 포인터 패턴** (정렬된 배열에서 쌍 찾기)
```go
func twoSum(nums []int, target int) (int, int) {
    l, r := 0, len(nums)-1
    for l < r {
        sum := nums[l] + nums[r]
        if sum == target { return l, r }
        if sum < target { l++ } else { r-- }
    }
    return -1, -1
}
```

**슬라이딩 윈도우** (연속 구간 최적화)
```go
// 길이 k인 최대 합 구간
func maxSum(nums []int, k int) int {
    sum, maxVal := 0, 0
    for i, v := range nums {
        sum += v
        if i >= k { sum -= nums[i-k] }
        if i >= k-1 && sum > maxVal { maxVal = sum }
    }
    return maxVal
}
```

---

## 3. 연결 리스트 (Linked List)

```
단방향: [data|next] → [data|next] → nil
이중:   nil ← [prev|data|next] ↔ [prev|data|next] → nil

접근: O(n)   삽입/삭제: O(1) (위치를 알 때)
```

**자주 나오는 문제 유형**
- 사이클 감지: 플로이드 토끼/거북이 알고리즘 (fast/slow 포인터)
- 역순: 포인터 3개로 in-place 반전
- 중간 노드: slow/fast 포인터

```go
// 사이클 감지
func hasCycle(head *ListNode) bool {
    slow, fast := head, head
    for fast != nil && fast.Next != nil {
        slow = slow.Next
        fast = fast.Next.Next
        if slow == fast { return true }
    }
    return false
}
```

---

## 4. 스택 & 큐

**스택 (LIFO)**: 재귀 → 반복 변환, 괄호 매칭, 히스토리(뒤로가기)
```go
stack := []int{}
stack = append(stack, val)       // push
val = stack[len(stack)-1]        // peek
stack = stack[:len(stack)-1]     // pop
```

**큐 (FIFO)**: BFS, 게임 이벤트 큐, 작업 스케줄링
```go
queue := []int{}
queue = append(queue, val)       // enqueue
val = queue[0]; queue = queue[1:]// dequeue (비효율적)
// 실제론 링 버퍼나 container/list 사용
```

**모노토닉 스택** (다음 큰 원소, 히스토그램 넓이)
```go
// 다음 큰 원소 인덱스
func nextGreater(nums []int) []int {
    result := make([]int, len(nums))
    stack := []int{}  // 인덱스 저장
    for i, v := range nums {
        for len(stack) > 0 && nums[stack[len(stack)-1]] < v {
            idx := stack[len(stack)-1]
            stack = stack[:len(stack)-1]
            result[idx] = i
        }
        stack = append(stack, i)
    }
    return result
}
```

---

## 5. 해시맵 (Hash Map)

```
평균: 삽입/삭제/조회 O(1)
최악: O(n) (해시 충돌)

충돌 해결:
- 체이닝: 버킷에 연결 리스트
- 오픈 어드레싱: 다음 빈 자리 탐색
```

**활용 패턴**
```go
// 빈도 카운트
freq := map[string]int{}
for _, word := range words { freq[word]++ }

// 두 배열 교집합
set := map[int]bool{}
for _, v := range a { set[v] = true }
for _, v := range b {
    if set[v] { /* 교집합 */ }
}

// LRU 캐시 → map + 이중 연결 리스트
```

---

## 6. 트리

**이진 탐색 트리 (BST)**
```
        8
       / \
      3   10
     / \    \
    1   6    14
       / \
      4   7

탐색/삽입/삭제: 평균 O(log n), 최악 O(n) (편향 트리)
```

**균형 트리**: AVL, Red-Black Tree → 항상 O(log n) 보장
- MySQL B-Tree 인덱스, Go의 `map`은 해시맵, Java TreeMap은 Red-Black Tree

**트리 순회**
```go
// 전위(Pre): Root → Left → Right
// 중위(In):  Left → Root → Right  ← BST에서 정렬 순서
// 후위(Post): Left → Right → Root

func inorder(node *TreeNode) {
    if node == nil { return }
    inorder(node.Left)
    fmt.Println(node.Val)
    inorder(node.Right)
}

// BFS (레벨 순회)
func bfs(root *TreeNode) {
    queue := []*TreeNode{root}
    for len(queue) > 0 {
        node := queue[0]; queue = queue[1:]
        fmt.Println(node.Val)
        if node.Left != nil { queue = append(queue, node.Left) }
        if node.Right != nil { queue = append(queue, node.Right) }
    }
}
```

---

## 7. 힙 (Heap)

```
최대 힙: 부모 ≥ 자식 (루트가 최댓값)
최소 힙: 부모 ≤ 자식 (루트가 최솟값)

삽입: O(log n)   추출: O(log n)   최솟값/최댓값 조회: O(1)
```

**활용**: 우선순위 큐, Top-K 문제, 다익스트라

```go
import "container/heap"

// Go 최소 힙
type MinHeap []int
func (h MinHeap) Len() int           { return len(h) }
func (h MinHeap) Less(i, j int) bool { return h[i] < h[j] }
func (h MinHeap) Swap(i, j int)      { h[i], h[j] = h[j], h[i] }
func (h *MinHeap) Push(x interface{}) { *h = append(*h, x.(int)) }
func (h *MinHeap) Pop() interface{} {
    n := len(*h); x := (*h)[n-1]; *h = (*h)[:n-1]; return x
}

h := &MinHeap{3, 1, 4, 1, 5}
heap.Init(h)
heap.Push(h, 2)
min := heap.Pop(h).(int)
```

---

## 8. 그래프

```
표현 방식:
- 인접 행렬: O(V²) 공간, 엣지 존재 확인 O(1)
- 인접 리스트: O(V+E) 공간, 이웃 순회 빠름

graph := map[int][]int{
    0: {1, 2},
    1: {3},
    2: {3},
    3: {},
}
```

**DFS** (깊이 우선 탐색)
```go
func dfs(node int, visited map[int]bool, graph map[int][]int) {
    if visited[node] { return }
    visited[node] = true
    fmt.Println(node)
    for _, next := range graph[node] {
        dfs(next, visited, graph)
    }
}
```

**BFS** (최단 경로, 레벨 탐색)
```go
func bfs(start int, graph map[int][]int) {
    visited := map[int]bool{start: true}
    queue := []int{start}
    for len(queue) > 0 {
        node := queue[0]; queue = queue[1:]
        fmt.Println(node)
        for _, next := range graph[node] {
            if !visited[next] {
                visited[next] = true
                queue = append(queue, next)
            }
        }
    }
}
```

**위상 정렬** (의존 관계 순서, DAG)
```
게임 퀘스트 선행 조건, 빌드 순서 결정 등에 활용
Kahn's Algorithm: 진입 차수가 0인 노드부터 큐에 넣어 처리
```

---

## 9. 정렬 알고리즘

| 알고리즘 | 평균 | 최악 | 공간 | 안정성 |
|---------|------|------|------|--------|
| 버블 | O(n²) | O(n²) | O(1) | ✅ |
| 선택 | O(n²) | O(n²) | O(1) | ❌ |
| 삽입 | O(n²) | O(n²) | O(1) | ✅ |
| 병합 | O(n log n) | O(n log n) | O(n) | ✅ |
| 퀵 | O(n log n) | O(n²) | O(log n) | ❌ |
| 힙 | O(n log n) | O(n log n) | O(1) | ❌ |
| 기수 | O(kn) | O(kn) | O(n+k) | ✅ |

**퀵 정렬 최악 회피**: 랜덤 피벗 선택 또는 Median-of-three

---

## 10. 동적 프로그래밍 (DP)

**핵심 개념**: 중복되는 하위 문제를 메모이제이션/테이블로 재사용

```go
// 피보나치 (Top-Down 메모이제이션)
memo := map[int]int{}
func fib(n int) int {
    if n <= 1 { return n }
    if v, ok := memo[n]; ok { return v }
    memo[n] = fib(n-1) + fib(n-2)
    return memo[n]
}

// 배낭 문제 (0/1 Knapsack) - Bottom-Up
func knapsack(weights, values []int, capacity int) int {
    dp := make([]int, capacity+1)
    for i := range weights {
        for c := capacity; c >= weights[i]; c-- {
            if dp[c-weights[i]]+values[i] > dp[c] {
                dp[c] = dp[c-weights[i]] + values[i]
            }
        }
    }
    return dp[capacity]
}
```

**자주 나오는 DP 유형**
- 최장 공통 부분 수열 (LCS)
- 최장 증가 부분 수열 (LIS)
- 동전 교환 (Coin Change)
- 격자 경로 수 계산

---

## 11. 이진 탐색

```go
// 정렬된 배열에서 target 인덱스
func binarySearch(nums []int, target int) int {
    l, r := 0, len(nums)-1
    for l <= r {
        mid := l + (r-l)/2  // 오버플로우 방지
        if nums[mid] == target { return mid }
        if nums[mid] < target { l = mid + 1 } else { r = mid - 1 }
    }
    return -1
}

// 응용: 정렬된 배열에서 조건을 만족하는 최솟값 탐색 (Parametric Search)
// "적어도 X 이상"인 최소 X를 이진 탐색으로 O(log n)에 찾기
```

---

## 12. 자주 나오는 면접 질문

**Q. 해시맵과 트리맵 차이?**
해시맵: 평균 O(1) 삽입/조회, 순서 없음. 트리맵(Red-Black Tree): O(log n), 키 정렬 순서 유지.

**Q. 스택으로 큐 구현?**
스택 2개 사용. 입력 스택에 push, 출력 스택이 비면 입력 스택 전체를 뒤집어 이동.

**Q. 힙과 정렬된 배열 중 우선순위 큐 구현에 뭐가 나은가?**
힙. 삽입 O(log n) vs 배열 삽입 O(n). 최솟값 추출은 둘 다 O(log n)이나 힙이 공간 효율 좋음.

**Q. DFS vs BFS 언제 선택?**
최단 경로 → BFS. 경로 존재 여부, 모든 경우 탐색, 깊은 구조 → DFS. 메모리 제한 있으면 DFS(스택).
