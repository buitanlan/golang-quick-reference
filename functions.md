# Hàm (Functions) trong Go

---

## Mục lục

1. [Khai báo hàm](#1-khai-báo-hàm)
2. [Nhiều giá trị trả về](#2-nhiều-giá-trị-trả-về)
3. [Named result parameters](#3-named-result-parameters)
4. [Variadic (`...T`)](#4-variadic-t)
5. [`defer` trong hàm](#5-defer-trong-hàm)
6. [Function values & first-class](#6-function-values--first-class)
7. [Closure](#7-closure)
8. [Đệ quy](#8-đệ-quy)
9. [`init`](#9-init)
10. [Blank identifier `_`](#10-blank-identifier-_)
11. [`panic` / `recover` (tóm tắt)](#11-panic--recover-tóm-tắt)
12. [Best practices](#12-best-practices)

---

## 1. Khai báo hàm

### 1.1 Cú pháp

```go
func Name(param1 Type1, param2 Type2) ReturnType {
    // body
    return value
}
```

Rút gọn khi cùng kiểu:

```go
func Add(a, b int) int {
    return a + b
}

func Swap(a, b string) (string, string) {
    return b, a
}
```

### 1.2 Tham số luôn truyền **theo giá trị**

Gán lại tham số không ảnh hưởng caller; với pointer/slice/map/chan, **header** được copy nhưng có thể cùng trỏ dữ liệu nền.

```go
func bump(p *int) {
    *p++
}

func appendLocal(s []int) {
    s = append(s, 99) // có thể không thấy ở caller nếu reallocate
}
```

### 1.3 Không overload, không default argument

Cùng tên trong cùng scope → lỗi. Muốn tùy chọn: dùng functional options, struct config, hoặc overload bằng tên khác (`Read`, `ReadAt`).

### 1.4 Package-level vs method

```go
func NewServer(addr string) *Server { /* ... */ }

func (s *Server) Start() error { /* method — xem methods-interfaces.md */ }
```

### 1.5 Main

```go
package main

func main() {
    // entry của executable; không tham số, không trả về
}
```

Đối với CLI: dùng `os.Args` hoặc thư viện `flag` / `cobra`.

---

## 2. Nhiều giá trị trả về

Idiom trung tâm của Go — đặc biệt **`(T, error)`**:

```go
func Div(a, b float64) (float64, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}

q, err := Div(10, 2)
if err != nil {
    log.Fatal(err)
}
fmt.Println(q)
```

Comma-ok với map / type assertion / channel nhận:

```go
v, ok := m[k]
v, ok := x.(string)
v, ok := <-ch
```

Bỏ giá trị không cần bằng `_`:

```go
_, err := os.Stat(path)
```

---

## 3. Named result parameters

```go
func SplitHostPort(hostport string) (host, port string, err error) {
    // host, port, err đã được zero-init
    i := strings.LastIndex(hostport, ":")
    if i < 0 {
        err = errors.New("missing port")
        return // naked return
    }
    host = hostport[:i]
    port = hostport[i+1:]
    return
}
```

Đặc điểm:

- Tên result là biến cục bộ của hàm.
- `return` không đối số (naked return) trả các giá trị hiện tại.
- `defer` có thể gán/sửa named result trước khi trả về thật.

```go
func ReadConfig(path string) (cfg Config, err error) {
    defer func() {
        if err != nil {
            err = fmt.Errorf("ReadConfig %s: %w", path, err)
        }
    }()
    // ...
    return cfg, err
}
```

**Khuyến nghị:** naked return chỉ trong hàm ngắn; hàm dài nên `return` tường minh.

---

## 4. Variadic (`...T`)

Tham số cuối có thể nhận 0+ đối số cùng kiểu:

```go
func Sum(nums ...int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}

Sum()
Sum(1, 2, 3)

xs := []int{4, 5}
Sum(xs...) // bung slice
```

Trong thân hàm, `nums` có kiểu `[]int`.

Kết hợp với interface:

```go
func Println(a ...any) (n int, err error) {
    return fmt.Println(a...)
}
```

Chỉ **một** variadic và phải ở **cuối** danh sách tham số.

---

## 5. `defer` trong hàm

```go
mu.Lock()
defer mu.Unlock()

resp, err := http.Get(url)
if err != nil {
    return err
}
defer resp.Body.Close()
```

### 5.1 Đánh giá đối số ngay

```go
func f() {
    i := 0
    defer fmt.Println(i) // in 0, dù i đổi sau
    i++
}
```

Muốn giá trị lúc chạy: closure đọc biến hoặc truyền pointer.

```go
defer func() { fmt.Println(i) }()
```

### 5.2 Thứ tự LIFO

```go
defer fmt.Println("first")  // chạy sau
defer fmt.Println("second") // chạy trước
```

### 5.3 Defer trong vòng lặp

Cẩn thận — defer gắn với **hàm**, không với lần lặp:

```go
// Sai nếu mở nhiều file trong loop mà chỉ Close khi hàm kết thúc
for _, path := range paths {
    f, err := os.Open(path)
    if err != nil {
        return err
    }
    defer f.Close() // dồn tất cả Close đến cuối hàm
}

// Đúng: bọc bằng hàm con
for _, path := range paths {
    if err := process(path); err != nil {
        return err
    }
}

func process(path string) error {
    f, err := os.Open(path)
    if err != nil {
        return err
    }
    defer f.Close()
    return read(f)
}
```

---

## 6. Function values & first-class

Hàm là giá trị first-class: gán biến, truyền đối số, trả về từ hàm.

```go
var op func(int, int) int
op = Add
fmt.Println(op(2, 3))

type Transform func(string) string

func Apply(xs []string, t Transform) []string {
    out := make([]string, len(xs))
    for i, x := range xs {
        out[i] = t(x)
    }
    return out
}

Apply(names, strings.ToUpper)
```

Literal (anonymous function):

```go
handler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintln(w, "ok")
})
```

So sánh: giá trị hàm chỉ so được với `nil`:

```go
var f func()
if f == nil {
    // chưa gán
}
```

---

## 7. Closure

Literal hàm capture biến từ scope ngoài:

```go
func makeCounter() func() int {
    n := 0
    return func() int {
        n++
        return n
    }
}

c := makeCounter()
fmt.Println(c()) // 1
fmt.Println(c()) // 2
```

### 7.1 Capture theo tham chiếu biến

```go
var funcs []func()
for i := 0; i < 3; i++ {
    i := i // bản sao mỗi vòng (cần trên Go < 1.22; 1.22+ biến lặp đã per-iteration)
    funcs = append(funcs, func() { fmt.Println(i) })
}
```

Go 1.22+ sửa semantic biến `for` — closure trong vòng lặp an toàn hơn theo mặc định.

### 7.2 Middleware / decorator

```go
func withLogging(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)
        log.Printf("%s %s %v", r.Method, r.URL.Path, time.Since(start))
    })
}
```

---

## 8. Đệ quy

```go
func Fact(n int) int {
    if n == 0 {
        return 1
    }
    return n * Fact(n-1)
}

func Walk(dir string, fn func(string) error) error {
    entries, err := os.ReadDir(dir)
    if err != nil {
        return err
    }
    for _, e := range entries {
        path := filepath.Join(dir, e.Name())
        if e.IsDir() {
            if err := Walk(path, fn); err != nil {
                return err
            }
            continue
        }
        if err := fn(path); err != nil {
            return err
        }
    }
    return nil
}
```

Go **không** tối ưu tail-call — đệ quy sâu có thể stack overflow; với cây lớn cân nhắc stack tường minh / queue.

---

## 9. `init`

```go
package config

var Default Timeout

func init() {
    Default = Timeout{Seconds: 30}
}
```

Đặc điểm:

- `func init()` không tham số, không trả về, không gọi tường minh.
- Một file có thể có nhiều `init`.
- Thứ tự: import dependencies → khởi tạo biến package theo thứ tự khai báo → các `init` theo thứ tự file (tool quyết định) rồi `main`.
- Dùng cho đăng ký driver (`_ "github.com/..."`), validate config lúc start.
- Tránh logic nặng / I/O phức tạp trong `init` — khó test và làm chậm startup.

```go
import _ "image/png" // init của image/png đăng ký decoder
```

---

## 10. Blank identifier `_`

`_` nhận giá trị rồi **bỏ** — không tạo binding dùng được.

```go
_ = must(os.Setenv("X", "1"))

for _, v := range xs {
    fmt.Println(v)
}

var _ http.Handler = (*MyHandler)(nil) // compile-time check interface
```

Import side-effect:

```go
import _ "net/http/pprof"
```

Trong assignment, `_` cho phép bỏ qua phần tử:

```go
a, _, c := triple()
```

---

## 11. `panic` / `recover` (tóm tắt)

- `panic(v)` dừng control flow bình thường, chạy defer, rồi crash process (trừ khi recover).
- `recover()` chỉ có tác dụng **bên trong defer**; trả giá trị panic hoặc `nil`.

```go
func safeRun(fn func()) (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("panic: %v", r)
        }
    }()
    fn()
    return nil
}
```

**Quy ước:**

- Lỗi thường gặp → trả `error`, **không** panic.
- Panic cho bug lập trình (index out of range, nil deref) hoặc trạng thái “không thể tiếp tục” thật sự.
- Library công cộng: tránh để panic xuyên ra caller; recover ở biên (HTTP middleware, worker).

> Chi tiết pattern lỗi, wrapping `%w`, `errors.Is`/`As`, sentinel: xem **errors.md** (khi có trong bộ tài liệu). Phần này chỉ overview `panic`/`recover` liên quan hàm.

---

## 12. Best practices

| Nên | Tránh |
|-----|--------|
| `(T, error)` rõ ràng | Panic cho lỗi I/O thường |
| Wrap lỗi với `%w` + ngữ cảnh | Naked return hàm dài |
| `defer Close()` ngay sau mở thành công | Ignore `err` bằng `_` im lặng |
| Tên hàm động từ rõ (`Parse`, `Load`) | Side-effect nặng trong `init` |
| Functional options cho API nhiều tùy chọn | Quá nhiều tham số positional |

### Ví dụ tổng hợp

```go
func LoadJSON[T any](path string) (T, error) {
    var zero T
    f, err := os.Open(path)
    if err != nil {
        return zero, fmt.Errorf("open %s: %w", path, err)
    }
    defer f.Close()

    var v T
    if err := json.NewDecoder(f).Decode(&v); err != nil {
        return zero, fmt.Errorf("decode %s: %w", path, err)
    }
    return v, nil
}
```

*(Generic cần Go 1.18+; tài liệu nhắm Go 1.21+.)*

---

## Tài liệu liên quan

- [statements.md](statements.md) — `defer`, `return`, scope
- [keywords.md](keywords.md) — `func`, `go`, `defer`, `return`
- [methods-interfaces.md](methods-interfaces.md) — method là hàm có receiver
- **errors.md** — xử lý lỗi chi tiết (tham chiếu)
