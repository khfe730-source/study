# Chapter 16. A web server

> 원문: [Effective Go — A web server](https://go.dev/doc/effective_go#web_server)

---

마지막은 완전한 Go 프로그램, **웹 서버**다. 정확히는 일종의 "웹 리서버(re-server)"다. Google이 `chart.apis.google.com`에서 데이터를 차트로 자동 포매팅하는 서비스를 제공하는데, 데이터를 URL 쿼리에 넣어야 해서 대화형으로 쓰기 어렵다. 이 프로그램은 더 나은 인터페이스를 제공한다 — **짧은 텍스트를 받아 차트 서버를 호출해 QR 코드를 생성**한다. 휴대폰 카메라로 찍어 URL 등으로 해석할 수 있다.

---

## 1. 전체 프로그램

```go
package main

import (
    "flag"
    "html/template"
    "log"
    "net/http"
)

var addr = flag.String("addr", ":1718", "http service address") // Q=17, R=18
var templ = template.Must(template.New("qr").Parse(templateStr))

func main() {
    flag.Parse()
    http.Handle("/", http.HandlerFunc(QR))
    err := http.ListenAndServe(*addr, nil)
    if err != nil {
        log.Fatal("ListenAndServe:", err)
    }
}

func QR(w http.ResponseWriter, req *http.Request) {
    templ.Execute(w, req.FormValue("s"))
}

const templateStr = `
<html>
<head>
<title>QR Link Generator</title>
</head>
<body>
{{if .}}
<img src="http://chart.apis.google.com/chart?chs=300x300&cht=qr&choe=UTF-8&chl={{.}}" />
<br>
{{.}}
<br>
<br>
{{end}}
<form action="/" name=f method="GET">
<input maxLength=1024 size=70 name=s value="" title="Text to QR Encode">
<input type=submit value="Show QR" name=qr>
</form>
</body>
</html>
`
```

---

## 2. 구조 해설

**`main`까지의 조각들** — 따라가기 쉽다.

- `addr` 플래그는 서버의 기본 HTTP 포트를 설정한다(`:1718`, Q=17·R=18의 말장난).
- `templ` 변수가 핵심이다 — 서버가 페이지를 표시하려고 실행할 HTML 템플릿을 만든다. `template.Must`는 파싱 에러가 있으면 panic하는 래퍼다(전역 초기화에서 panic이 합당한 예 — 14장 Errors 참고).

**`main` 함수**는 플래그를 파싱하고, 앞서 본 메커니즘(11장 Interfaces의 `HandlerFunc`)으로 **함수 `QR`을 루트 경로(`/`)에 바인딩**한다. 그 뒤 `http.ListenAndServe`로 서버를 시작하며, 서버가 도는 동안 블록된다.

```go
http.Handle("/", http.HandlerFunc(QR)) // 함수를 핸들러로 변환해 바인딩
```

**`QR` 함수**는 폼 데이터를 담은 요청을 받아, 폼 값 `s`에 대해 템플릿을 실행한다.

```go
func QR(w http.ResponseWriter, req *http.Request) {
    templ.Execute(w, req.FormValue("s"))
}
```

---

## 3. html/template 핵심

`html/template` 패키지는 강력하다(이 프로그램은 일부만 건드린다). 본질적으로 **`templ.Execute`에 전달된 데이터 항목에서 파생된 요소를 치환해 HTML 텍스트를 즉석에서 재작성**한다.

템플릿 텍스트에서 **이중 중괄호(`{{ }}`)로 둘러싼 부분이 템플릿 액션**이다.

- `{{if .}}` ~ `{{end}}` — 현재 데이터 항목 `.`(dot)이 **비어 있지 않을 때만** 실행. 문자열이 비면 이 부분이 억제된다.
- `{{.}}` — 템플릿에 제공된 데이터(쿼리 문자열)를 페이지에 표시.

> **자동 이스케이핑(escaping):** HTML 템플릿 패키지는 적절한 이스케이핑을 자동 제공해 텍스트를 안전하게 표시한다 — XSS 등 인젝션을 방지한다. (이것이 `text/template`이 아니라 `html/template`을 쓰는 이유다.)

나머지 템플릿 문자열은 페이지 로드 시 보여줄 평범한 HTML이다.

---

## 4. 정리

| 요소 | 역할 |
|---|---|
| `flag` | `addr`로 포트 설정, `flag.Parse()` |
| `template.Must` | 파싱 실패 시 panic하는 초기화 래퍼 |
| `http.HandlerFunc(QR)` | 일반 함수를 핸들러로 변환해 `/`에 바인딩 |
| `http.ListenAndServe` | 서버 시작, 실행 중 블록 |
| `html/template` | 데이터 기반 HTML 재작성 + **자동 이스케이핑** |

**한 줄 요약:** 몇 줄의 코드와 데이터 기반 HTML 템플릿만으로 유용한 웹 서버를 만들 수 있다 — `HandlerFunc`로 함수를 핸들러화하고, `html/template`이 안전한 이스케이핑과 함께 데이터를 페이지에 주입한다. Go는 적은 코드로 많은 일을 해낼 만큼 강력하다.
