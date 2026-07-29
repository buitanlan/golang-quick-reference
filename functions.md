# Hàm (Functions) trong Go

Hàm là đơn vị tổ chức code cơ bản của Go, đồng thời là **giá trị first-class**. Nắm chắc bốn thứ là đủ tự tin: thời điểm đối số được đánh giá, named result tương tác với `defer`/`recover` thế nào, closure capture cái gì, và khi nào compiler inline / đẩy dữ liệu lên heap.

> Tài liệu nhắm **Go 1.26**; tính năng mới hơn 1.21 được ghi rõ version — xem bảng ở cuối. Luật kiểu nào được bật phụ thuộc directive `go` trong `go.mod`, không phải toolchain đang cài — xem [typesystem.md](typesystem.md) và [build-constraints.md](build-constraints.md).

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
11. [`panic` / `recover`](#11-panic--recover)
12. [Method value & method expression](#12-method-value--method-expression)
13. [Hàm generic & suy luận kiểu](#13-hàm-generic--suy-luận-kiểu)
14. [Kiểu hàm & iterator (Go 1.23+)](#14-kiểu-hàm--iterator-go-123)
15. [Inlining, escape & chi phí gọi hàm](#15-inlining-escape--chi-phí-gọi-hàm)
16. [Best practices](#16-best-practices)

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

Go **chỉ** có pass-by-value — không có reference parameter như C++ `&`. Gán lại tham số không ảnh hưởng caller; với pointer/slice/map/chan, **header** được copy nhưng có thể cùng trỏ dữ liệu nền.

```go
func bump(p *int) {
    *p++ // sửa được vì p là bản sao của con trỏ, cùng trỏ tới biến gốc
}

func appendLocal(s []int) {
    s[0] = 1          // caller THẤY: cùng backing array
    s = append(s, 99) // caller KHÔNG thấy: gán lại header cục bộ
}
```

Chi phí copy = `unsafe.Sizeof` của kiểu tham số (shallow):


| Kiểu tham số     | Byte copy mỗi lần gọi (amd64) |
| ---------------- | ----------------------------- |
| `int`, pointer   | 8                             |
| `string`         | 16 (ptr + len)                |
| slice            | 24 (ptr + len + cap)          |
| `any` / interface | 16 (type + data)             |
| `struct{...}`    | tổng size sau padding         |


- Struct lớn (vài trăm byte trở lên) trong hot path → nhận `*T` để tránh copy; xem §15 và [pointers.md](pointers.md).
- Ngược lại, `*T` cho struct nhỏ có thể **chậm hơn**: thêm một lần indirection và dễ làm giá trị escape lên heap.

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

### 2.1 Nhiều result vs trả struct

Multi-result là nhóm giá trị **không có tên ở phía caller**; struct có tên và mở rộng được mà không phá API.


| Chọn                          | Khi                                                                              |
| ----------------------------- | -------------------------------------------------------------------------------- |
| `(T, error)`                  | Mặc định — một kết quả + lỗi                                                     |
| `(T, bool)`                   | Comma-ok: “không tìm thấy” **không** phải lỗi (`m[k]`, `x.(T)`, `<-ch`)           |
| 3 result trở lên              | Chỉ khi các giá trị thật sự đồng hạng (`host, port, err`)                         |
| `(Result, error)` với `struct` | Từ 3 giá trị trở lên, hoặc còn có thể thêm field sau này, hoặc nhiều giá trị cùng kiểu dễ đổi chỗ |


```go
// Khó đọc và dễ gọi sai: 4 giá trị, 3 trong đó cùng kiểu
func Stat(p string) (size int64, mode int64, mtime int64, err error)

// Rõ ràng, thêm field sau này không phá caller
type FileInfo struct {
    Size  int64
    Mode  fs.FileMode
    MTime time.Time
}

func Stat(p string) (FileInfo, error)
```

- Thêm result vào hàm exported là **breaking change**; thêm field vào struct result thì không.
- Trả `struct` theo **giá trị** (không phải `*Result`) nếu nhỏ — tránh allocation không cần thiết.
- Đừng trả `(T, bool, error)`: chọn một trong hai cách báo “không có”.

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

- Tên result là **biến cục bộ** của hàm, đã zero-init khi vào hàm.
- `return` không đối số (naked return) trả các giá trị hiện tại.
- `defer` có thể gán/sửa named result trước khi trả về thật.

### 3.1 Thứ tự chính xác: `return` → gán result → `defer` → về caller

Spec nói rõ: “deferred functions are executed **after any result parameters are set** by that return statement but before the function returns to its caller”. Vì vậy `return expr` với named result không phải là “trả ngay”:

```go
func double() (n int) {
    defer func() { n *= 2 }()
    return 21 // gán n = 21, chạy defer (n = 42), rồi trả 42
}

func doubleUnnamed() int {
    n := 21
    defer func() { n *= 2 }() // sửa biến cục bộ, KHÔNG sửa được result
    return n                  // trả 21
}
```

Đây là lý do **mọi** pattern “defer sửa lỗi trả về” đều bắt buộc named result:

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

Hai ứng dụng chính:

```go
// 1) Biến panic thành error ở biên package
func safeRun(fn func()) (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("panic: %v", r)
        }
    }()
    fn()
    return nil
}

// 2) Không đánh mất lỗi của Close() khi ghi file
func writeFile(name string, data []byte) (err error) {
    f, err := os.Create(name)
    if err != nil {
        return err
    }
    defer func() {
        if cerr := f.Close(); cerr != nil && err == nil {
            err = cerr // flush thất bại chỉ lộ ra ở Close
        }
    }()
    _, err = f.Write(data)
    return err
}
```

> Với **writer**, `defer f.Close()` trần là bug tiềm ẩn: lỗi flush bị bỏ im lặng. Với **reader** thì `defer f.Close()` là đủ.

### 3.2 Bẫy shadowing

`if err := ...` tạo một `err` **mới** che named result. `return err` tường minh vẫn gán vào result nên vẫn đúng; nguy hiểm là nhánh **không** return:

```go
func broken() (err error) {
    defer func() {
        if err != nil {
            err = fmt.Errorf("wrapped: %w", err)
        }
    }()
    if err := doWork(); err != nil { // err này chỉ sống trong if
        log.Println(err)             // quên "return err"
    }
    return // named result vẫn nil → defer không wrap gì, lỗi bị nuốt
}
```

Naked return **trong** scope bị che thì compiler từ chối thẳng (đúng “Implementation restriction” của spec):

```go
func caseC() (err error) {
    if err := doWork(); err != nil {
        return // lỗi biên dịch
    }
    return
}
```

Output thật của `go build` (go1.26.5):

```text
./shadow.go:41:3: result parameter err not in scope at return
	./shadow.go:39:5: inner declaration of var err error
```

- Nhánh “quên return” thì **không** có lỗi biên dịch và `go vet` mặc định cũng **không** bắt — cần `shadow` analyzer hoặc `golangci-lint`.
- Cách phòng đơn giản nhất: dùng `err = doWork()` (không `:=`) trong hàm đã có named result `err`.

### 3.3 Khi nào dùng named result

| Nên | Tránh |
|-----|--------|
| Cần `defer` sửa result (wrap lỗi, recover, Close) | Hàm dài rồi naked return ở nhiều nhánh |
| Tên làm rõ result cùng kiểu: `(host, port string, err error)` | Đặt tên chỉ để “trông chuyên nghiệp” rồi vẫn `return a, b` |
| Tài liệu hóa signature trong godoc | Dựa vào zero-init thay vì gán tường minh |

**Khuyến nghị:** đặt tên cho result nhưng vẫn `return` **tường minh**; naked return chỉ trong hàm rất ngắn.

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

Trong thân hàm, `nums` có kiểu `[]int`. Chỉ **một** variadic và phải ở **cuối** danh sách tham số.

### 4.1 Hai cách truyền — chỉ một cách copy

Spec phân biệt rõ hai trường hợp, và đây là chỗ hay gây bug:


| Cách gọi         | Giá trị nhận được trong hàm                                   |
| ---------------- | ------------------------------------------------------------- |
| `Sum()`          | `nil` — **không** phải `[]int{}`                               |
| `Sum(1, 2, 3)`   | slice **mới**, backing array mới → callee không sửa được caller |
| `Sum(xs...)`     | **chính** `xs`, không tạo slice mới → callee sửa được caller!  |


```go
func clobber(xs ...int) { xs[0] = -1 }

xs := []int{1, 2, 3}
clobber(xs...)
fmt.Println(xs) // [-1 2 3] — bị sửa

func isNil(nums ...int) bool { return nums == nil }
fmt.Println(isNil())        // true
fmt.Println(isNil([]int{}...)) // false — slice rỗng khác nil
```

- Hàm variadic **được phép** giữ hay sửa slice → nếu API của bạn không mong điều đó, ghi vào doc comment hoặc `slices.Clone` trước khi lưu.
- `len(nums)` luôn dùng được (`len(nil) == 0`), nên hiếm khi cần phân biệt nil/rỗng — trừ khi ngữ nghĩa “không truyền gì” khác “truyền danh sách rỗng”.

### 4.2 `append` và forwarding

```go
s = append(s, other...)          // bung slice
s = append(s, "abc"...)          // chỉ hợp lệ khi s là []byte
dst = append(dst[:0], src...)    // reuse backing array
```

Forwarding phải giữ nguyên `...`:

```go
func Logf(format string, args ...any) {
    log.Printf(format, args...)  // đúng
    // log.Printf(format, args)  // sai: truyền 1 đối số kiểu []any → "%!d(MISSING)"
}
```

`go vet` bắt lỗi này qua analyzer `printf`: `missing ... in args forwarded to printf-like function`.

### 4.3 `[]T` không tự thành `...any`

Không có covariance: chỉ `[]any` mới bung được vào `...any`.

```go
names := []string{"a", "b"}
// fmt.Println(names...)
// LỖI: cannot use names (variable of type []string) as []any value in argument to fmt.Println

args := make([]any, len(names))
for i, n := range names {
    args[i] = n
}
fmt.Println(args...) // OK — phải convert từng phần tử
```

Đây là một trong những lý do generics tồn tại: `func Join[T any](xs []T)` nhận trực tiếp `[]string` mà không cần boxing — xem §13 và [generics.md](generics.md).

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

Câu lệnh `defer` đánh giá **cả function value lẫn toàn bộ đối số** ngay tại chỗ; chỉ *lời gọi* bị hoãn.

```go
func f() {
    i := 0
    defer fmt.Println(i) // in 0, dù i đổi sau
    i++
}
```

Muốn giá trị lúc chạy: closure đọc biến hoặc truyền pointer.

```go
defer func() { fmt.Println(i) }() // in 1 — closure đọc i khi chạy
```

Hệ quả tinh vi khác: **receiver của method value cũng bị chốt ngay**.

```go
var buf bytes.Buffer
defer fmt.Println(buf.Len())   // 0 — gọi Len() NGAY, chỉ hoãn Println
defer buf.WriteString("late")  // WriteString hoãn, nhưng receiver &buf chốt ngay
```

Và `defer` một function value `nil` chỉ panic **khi chạy**, không panic tại câu `defer` (spec):

```go
var cleanup func()
defer cleanup() // câu defer OK, panic khi hàm kết thúc
```

### 5.2 Thứ tự LIFO

```go
defer fmt.Println("first")  // chạy sau
defer fmt.Println("second") // chạy trước
```

Đầy đủ thứ tự khi hàm kết thúc bằng `return expr` và có named result:

```text
1. đánh giá expr
2. gán vào (named) result parameters
3. chạy các deferred function theo LIFO  ← chỗ duy nhất còn sửa được result
4. trả về caller
```

`defer` cũng chạy khi hàm thoát vì `panic` — đó là nền tảng của `recover` (§11) và của việc unlock mutex an toàn.

### 5.3 Chi phí `defer`

Từ **Go 1.14**, compiler dùng *open-coded defer*: theo release notes, `defer` “incur almost zero overhead compared to calling the deferred function directly”. Nghĩa là **không** cần tránh `defer` vì performance nữa. Hai ngoại lệ:

- `defer` trong vòng lặp: mỗi lần lặp đẩy thêm một entry, chạy dồn ở cuối hàm (§5.4) — vấn đề đúng đắn, không phải vi mô.
- `defer` làm hàm **không inline được**. Output thật của `go build -gcflags='-m -m'` (go1.26.5):

```text
./noinl.go:5:6:  cannot inline withDefer: unhandled op DEFER
./noinl.go:10:8: cannot inline withRecover.func1: call to recover
```

Vậy nên với accessor cực nhỏ trong hot path, `defer` có thể chặn inline — xem §15.

### 5.4 Defer trong vòng lặp

Cẩn thận — defer gắn với **hàm**, không với block hay lần lặp:

```go
// SAI: dồn tất cả Close đến cuối hàm → có thể vượt giới hạn file descriptor
func processAll(paths []string) error {
    for _, path := range paths {
        f, err := os.Open(path)
        if err != nil {
            return err
        }
        defer f.Close()
        if err := read(f); err != nil {
            return err
        }
    }
    return nil
}
```

Ba cách sửa, theo thứ tự ưu tiên:

```go
// 1) Tách thành hàm có tên — dễ test, dễ đọc nhất
func processAll(paths []string) error {
    for _, path := range paths {
        if err := processOne(path); err != nil {
            return fmt.Errorf("%s: %w", path, err)
        }
    }
    return nil
}

func processOne(path string) error {
    f, err := os.Open(path)
    if err != nil {
        return err
    }
    defer f.Close()
    return read(f)
}
```

```go
// 2) IIFE khi thân xử lý quá ngắn để tách hàm
for _, path := range paths {
    if err := func() error {
        f, err := os.Open(path)
        if err != nil {
            return err
        }
        defer f.Close()
        return read(f)
    }(); err != nil {
        return err
    }
}
```

```go
// 3) Close tường minh, KHÔNG defer — chỉ khi mọi nhánh lỗi đều đã Close
for _, path := range paths {
    f, err := os.Open(path)
    if err != nil {
        return err
    }
    err = read(f)
    f.Close() // dễ quên ở một nhánh return nào đó
    if err != nil {
        return err
    }
}
```

- `defer` trong `for`/`if`/`switch` block vẫn chỉ chạy khi **hàm** kết thúc — không có scope-based destructor như C++ RAII.
- Ngược lại, `defer mu.Unlock()` ngay sau `mu.Lock()` **là** idiom đúng, kể cả khi hàm dài: xem [concurrency.md](concurrency.md).

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

### 6.1 Kiểu hàm **không** comparable

Giá trị hàm chỉ so được với `nil`; `f == g` là lỗi biên dịch, và kiểu hàm không dùng được làm map key.

```go
var f func()
if f == nil { // OK
    // chưa gán
}

// if f == g {}          // LỖI: func can only be compared to nil
// _ = map[func()]bool{} // LỖI: invalid map key type func()
```

- Đây cũng là lý do struct chứa field kiểu hàm mất tính comparable — xem [structs-composition.md](structs-composition.md).
- Muốn “so sánh” handler cho mục đích debug: dùng `reflect.ValueOf(f).Pointer()` (chỉ để log, hai closure khác nhau của cùng một literal có thể cùng pointer).

### 6.2 Adapter: biến hàm thành interface

Conversion sang **named function type** là cách Go cho một hàm trần thỏa interface:

```go
type HandlerFunc func(http.ResponseWriter, *http.Request)

func (f HandlerFunc) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    f(w, r) // method trên kiểu hàm, receiver là chính hàm đó
}
```

Stdlib dùng pattern này ở `http.HandlerFunc` (thỏa `http.Handler`) và `expvar.Func` — `type Func func() any` với method `String()`/`Value()` để thỏa `expvar.Var`. Xem thêm [methods-interfaces.md](methods-interfaces.md).

### 6.3 Wrapper hàm hữu ích trong stdlib


| API                                   | Version | Dùng cho                                              |
| ------------------------------------- | ------- | ----------------------------------------------------- |
| `sync.OnceFunc(f func()) func()`      | 1.21    | Khởi tạo lazy, an toàn goroutine, thay `sync.Once` thủ công |
| `sync.OnceValue[T](f func() T) func() T` | 1.21 | Lazy singleton có giá trị trả về                       |
| `sync.OnceValues[T1,T2]`              | 1.21    | Lazy `(T, error)`                                      |
| `context.AfterFunc(ctx, f) (stop func() bool)` | 1.21 | Chạy `f` khi ctx xong; `stop` để hủy đăng ký    |
| `cmp.Or(vals ...T) T`                 | 1.22    | Lấy giá trị non-zero đầu tiên — thay chuỗi `if x == ""` |


```go
var loadConfig = sync.OnceValue(func() Config {
    return mustLoad("config.yaml") // chạy đúng một lần, kể cả nhiều goroutine
})

addr := cmp.Or(flagAddr, os.Getenv("ADDR"), ":8080")
```

> `sync.OnceValue` panic lại **cùng giá trị** ở mọi lần gọi nếu `f` panic — không “thử lại”.

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

Closure capture **biến**, không phải giá trị: `n` sống lâu hơn `makeCounter` nên compiler đẩy nó lên heap. Output thật `go build -gcflags='-m'` (go1.26.5):

```text
./main.go:30:2: moved to heap: n
./main.go:31:9: func literal escapes to heap
```

### 7.1 Biến vòng lặp: per-iteration từ Go 1.22

Trước 1.22, **một** biến `i` duy nhất được chia sẻ cho cả vòng lặp, nên mọi closure thấy giá trị cuối:

```go
// Go < 1.22: in "3 3 3"
var funcs []func()
for i := 0; i < 3; i++ {
    funcs = append(funcs, func() { fmt.Println(i) })
}
for _, f := range funcs {
    f()
}
```

Cách sửa cổ điển là tự tạo bản sao mỗi vòng:

```go
for i := 0; i < 3; i++ {
    i := i // "i := i" — nay không cần nữa
    funcs = append(funcs, func() { fmt.Println(i) })
}
```

Từ **Go 1.22**, biến khai báo trong `for ... := ...` (cả dạng 3-clause và `range`) là **per-iteration**, nên đoạn đầu in `0 1 2`. Ba lưu ý:

- Bật/tắt theo **language version của file**, tức directive `go` trong `go.mod` (hoặc `//go:build go1.21` cho từng file) — cài Go 1.26 mà `go.mod` ghi `go 1.21` thì vẫn dùng luật cũ.
- Chỉ áp dụng cho biến **khai báo bởi chính** `for`. `for i = range n` (gán, không `:=`) vẫn dùng biến ngoài, chia sẻ như trước.
- Đổi lại có thể có thêm allocation: nếu biến escape, mỗi vòng cấp phát một biến mới. Đo bằng `-benchmem` trước khi lo.

`go vet` có analyzer `loopclosure` bắt các trường hợp rõ ràng, nhưng chỉ trên code có language version < 1.22.

### 7.2 Closure giữ tham chiếu — bẫy rò rỉ

Closure giữ **toàn bộ** biến nó capture sống, kể cả khi bạn chỉ cần một field nhỏ:

```go
func handler(req *HugeRequest) func() {
    return func() { log.Println(req.ID) } // giữ cả HugeRequest sống
}

func handlerBetter(req *HugeRequest) func() {
    id := req.ID                          // chỉ giữ string
    return func() { log.Println(id) }
}
```

- Cùng logic với `runtime.AddCleanup` (1.24): callback **không được** capture object đang theo dõi, nếu không nó không bao giờ được thu hồi — xem [typesystem.md](typesystem.md).

### 7.3 Middleware / decorator

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

Go **không** tối ưu tail-call. Stack goroutine tự lớn dần (runtime copy sang stack lớn hơn), nhưng có trần: `runtime/debug.SetMaxStack` cho biết mặc định là **1 GB trên 64-bit, 250 MB trên 32-bit**; vượt trần thì process crash (`goroutine stack exceeds ... limit`), **không** recover được.

- Cây/graph sâu không giới hạn (parse input từ ngoài) → dùng stack/queue tường minh, hoặc đếm depth và trả lỗi.
- Đệ quy vẫn inline được **một lần**; sâu hơn thì compiler dừng. Output thật `-gcflags='-m -m'`:

```text
./noinl.go:92:13: cannot inline recurse into callAll: repeated recursive cycle
```

- Hàm đệ quy trên cây thư mục: ưu tiên `filepath.WalkDir` (`io/fs.WalkDirFunc`) thay vì tự viết — nó đã xử lý symlink, lỗi từng entry và không đệ quy sâu vô hạn.

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

- `func init()` không tham số, không trả về, **không thể gọi hay tham chiếu** tường minh — spec nói identifier `init` chỉ dùng để khai báo init function và bản thân nó không được declare.
- Một file có thể có nhiều `init`; một package có thể có nhiều file cùng khai báo `init`.
- Dùng cho đăng ký driver (`_ "github.com/..."`), validate config lúc start.
- Tránh logic nặng / I/O phức tạp trong `init` — khó test, không báo lỗi được (chỉ panic) và làm chậm startup.

```go
import _ "image/png" // init của image/png đăng ký decoder
```

### 9.1 Thứ tự khởi tạo chính xác

1. **Package**: sắp mọi package theo import path; mỗi bước khởi tạo package đầu tiên chưa init mà **mọi import đã init xong**. Mỗi package init đúng một lần.
2. **Biến package-level**: theo **dependency order**, *không* phải thứ tự khai báo. Mỗi bước chọn biến **sớm nhất theo thứ tự khai báo** trong số các biến đã sẵn sàng (không còn phụ thuộc biến chưa khởi tạo).
3. **`init` function**: theo thứ tự xuất hiện trong source, có thể xuyên nhiều file, theo thứ tự file được đưa cho compiler. `go build` đưa file theo thứ tự tên file (lexical) — nhưng spec chỉ **khuyến nghị** điều đó, nên đừng viết code phụ thuộc thứ tự giữa các file.
4. Cuối cùng `main.main`.

> Điểm dễ nhớ sai: biến package **không** khởi tạo theo thứ tự khai báo mà theo phụ thuộc. Ví dụ dưới đây `a` chạy trước `x` dù khai báo sau:

```go
var x = a + 1 // x phụ thuộc a → khởi tạo SAU
var a = f()   // không phụ thuộc gì → khởi tạo TRƯỚC
```

- Nhiều biến ở LHS của một khai báo multi-value được khởi tạo **cùng một bước**: `var a, b = f()`.
- Biến `_` cũng tham gia dependency analysis như biến thường — nên `var _ = mustInit()` là cách chạy code lúc init.
- Vòng phụ thuộc giữa các biến → **lỗi biên dịch** (`initialization cycle`).
- Toàn bộ quá trình chạy trong **một goroutine, tuần tự**: `init` có thể `go` ra goroutine khác, nhưng runtime không gọi `init` tiếp theo trước khi `init` hiện tại return.

### 9.2 Thay `init` bằng gì

| Vấn đề của `init` | Thay bằng |
|-------------------|-----------|
| Không trả được `error` → chỉ panic | `func New...() (*T, error)` gọi từ `main` |
| Chạy dù test không cần | `sync.OnceValue` (lazy, §6.3) |
| Thứ tự ngầm giữa nhiều file | Truyền dependency tường minh |
| Khó mock trong test | Biến package thành field của struct |

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

## 11. `panic` / `recover`

- `panic(v)` dừng control flow bình thường, chạy defer của frame hiện tại rồi lan lên caller, cuối cùng crash process (trừ khi recover).
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

### 11.1 `recover` phải được **deferred function gọi trực tiếp**

Spec: “The return value of `recover` is `nil` when the goroutine is not panicking **or `recover` was not called directly by a deferred function**.” Bọc thêm một lớp hàm là mất tác dụng:

```go
// SAI: recover được swallow gọi, không phải hàm được defer
func broken() (r any) {
    swallow := func() { r = recover() }
    defer func() { swallow() }() // r vẫn nil, panic tiếp tục lan
    panic("boom")
}

// ĐÚNG: recover nằm ngay trong thân hàm được defer
func ok() (r any) {
    defer func() { r = recover() }()
    panic("boom")
}
```

Vì thế **không** viết được helper kiểu `func Recover(err *error) { if r := recover(); ... }` rồi `defer Recover(&err)`... — trừ khi chính helper đó là hàm được defer:

```go
// ĐÚNG: helper CHÍNH LÀ hàm được defer
func recoverTo(err *error) {
    if r := recover(); r != nil {
        *err = fmt.Errorf("panic: %v", r)
    }
}

func run() (err error) {
    defer recoverTo(&err) // recover được gọi trực tiếp bởi recoverTo
    return doRisky()
}
```

### 11.2 `panic(nil)` → `*runtime.PanicNilError` (Go 1.21+)

Trước 1.21, `panic(nil)` làm `recover()` trả `nil`, khiến `if r := recover(); r != nil` bỏ lọt panic. Từ **Go 1.21**, runtime thay bằng `*runtime.PanicNilError`:

```go
func p() (r any) {
    defer func() { r = recover() }()
    panic(nil)
}
// Go 1.21+: r là *runtime.PanicNilError, r != nil
// Go < 1.21: r == nil
```

- Quay lại hành vi cũ: `GODEBUG=panicnil=1`.
- Spec diễn đạt cùng ý: gọi `panic` với nil interface (hoặc untyped `nil`) “causes a run-time panic”.

### 11.3 Cái gì recover được, cái gì không

| Loại | Recover được? |
|------|---------------|
| `panic(v)` tường minh | Có |
| Runtime error (`index out of range`, nil deref, chia 0, ghi nil map, type assertion sai) | Có — giá trị thỏa `runtime.Error` |
| Panic trong **goroutine khác** | Không — mỗi goroutine phải tự recover |
| `fatal error: concurrent map writes` / `all goroutines are asleep - deadlock!` | Không — fatal error bỏ qua defer |
| `os.Exit`, `log.Fatal` | Không — không chạy defer |
| Stack vượt trần (`goroutine stack exceeds`) | Không |

```go
// Panic trong goroutine giết cả process, dù caller có recover
go func() {
    defer func() { recover() }() // BẮT BUỘC ở đây, không phải ở hàm sinh ra goroutine
    work()
}()
```

Phân biệt panic của bạn với runtime error:

```go
defer func() {
    switch r := recover().(type) {
    case nil:
        // không panic
    case runtime.Error:
        // bug thật (nil deref, out of range) — log stack rồi để chết
        log.Printf("runtime error: %v\n%s", r, debug.Stack())
        panic(r) // re-panic
    default:
        err = fmt.Errorf("%v", r)
    }
}()
```

### 11.4 Re-panic và stack trace

`panic(r)` trong deferred function tạo một panic **mới**, nên stack trace in ra trỏ về chỗ re-panic, không phải chỗ panic gốc. Muốn giữ ngữ cảnh gốc: gọi `debug.Stack()` **ngay trong** deferred function (trước khi re-panic) và log lại.

**Quy ước:**

- Lỗi thường gặp → trả `error`, **không** panic.
- Panic cho bug lập trình hoặc trạng thái “không thể tiếp tục” thật sự (config sai lúc `init`, invariant bị phá).
- Library công cộng: tránh để panic xuyên ra caller; recover ở biên (HTTP middleware, worker pool).
- `net/http` đã tự recover panic của handler: theo doc của `http.Handler`, server “recovers the panic, logs a stack trace to the server error log, and either closes the network connection or sends an HTTP/2 RST_STREAM”. Muốn cắt response mà **không** ghi log lỗi: `panic(http.ErrAbortHandler)`.
- `Must...` prefix là quy ước cho hàm panic khi lỗi, chỉ dùng ở init/test: `regexp.MustCompile`, `template.Must`.

> Chi tiết pattern lỗi, wrapping `%w`, `errors.Is`/`As`/`AsType`, sentinel: xem [errors.md](errors.md).

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

*(Generic cần Go 1.18+; tài liệu nhắm Go 1.26.)*

---

## Tài liệu liên quan

- [statements.md](statements.md) — `defer`, `return`, scope
- [keywords.md](keywords.md) — `func`, `go`, `defer`, `return`
- [methods-interfaces.md](methods-interfaces.md) — method là hàm có receiver
- **errors.md** — xử lý lỗi chi tiết (tham chiếu)
