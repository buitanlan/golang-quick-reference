# Từ khóa (Keywords) trong Go

---

## Mục lục

1. [`break`](#1-break)
2. [`case`](#2-case)
3. [`chan`](#3-chan)
4. [`const`](#4-const)
5. [`continue`](#5-continue)
6. [`default`](#6-default)
7. [`defer`](#7-defer)
8. [`else`](#8-else)
9. [`fallthrough`](#9-fallthrough)
10. [`for`](#10-for)
11. [`func`](#11-func)
12. [`go`](#12-go)
13. [`goto`](#13-goto)
14. [`if`](#14-if)
15. [`import`](#15-import)
16. [`interface`](#16-interface)
17. [`map`](#17-map)
18. [`package`](#18-package)
19. [`range`](#19-range)
20. [`return`](#20-return)
21. [`select`](#21-select)
22. [`struct`](#22-struct)
23. [`switch`](#23-switch)
24. [`type`](#24-type)
25. [`var`](#25-var)
26. [Predeclared identifiers (không phải keyword)](#26-predeclared-identifiers-không-phải-keyword)
27. [Ngữ nghĩa keyword theo phiên bản & modernizer](#27-ngữ-nghĩa-keyword-theo-phiên-bản--modernizer)

Go có đúng **25 từ khóa dành riêng (reserved keywords)**. Danh sách này **chưa từng thay đổi** kể từ Go 1 — mọi tính năng mới (generics, `range` over int/func, `new(expr)`) đều được thêm mà không cần keyword mới. Không dùng chúng làm tên biến, hàm, kiểu, hay package.

---

## 1. `break`

- **Mục đích:** Thoát khỏi vòng lặp (`for`) hoặc `switch`/`select` gần nhất; có thể kèm **label** để thoát khối ngoài.

**Ví dụ:**

```go
for i := 0; i < 100; i++ {
    if i == 10 {
        break
    }
}

Outer:
for i := 0; i < 3; i++ {
    for j := 0; j < 3; j++ {
        if i*j > 2 {
            break Outer
        }
    }
}
```

**Ghi chú:**

- `break` không nhãn chỉ ảnh hưởng `for`/`switch`/`select` bao quanh **gần nhất**. Hệ quả quan trọng: `break` bên trong `switch`/`select` nằm trong `for` chỉ thoát `switch`/`select`, **không** thoát vòng lặp — muốn thoát vòng phải dùng label.
- Label của `break` phải gắn trực tiếp lên một `for`/`switch`/`select` bao quanh. Gắn lên `if` thì compiler báo `invalid break label L`.
- Trong `select`, `break` chỉ kết thúc case đang chạy và ra khỏi `select` (không "thử case khác").

---

## 2. `case`

- **Mục đích:** Nhánh trong `switch` hoặc `select`. Trong `switch`, các `case` không “rơi” xuống nhau trừ khi dùng `fallthrough`.

**Ví dụ:**

```go
switch day {
case 1, 2, 3, 4, 5:
    fmt.Println("weekday")
case 6, 7:
    fmt.Println("weekend")
}

select {
case v := <-ch:
    fmt.Println(v)
case ch <- 1:
    // gửi được
}
```

**Ghi chú:**

- Một `case` có thể liệt kê nhiều biểu thức (cách nhau bằng dấu phẩy). Trong `select`, mỗi `case` là một thao tác channel.
- Trùng hằng số trong cùng expression switch là **lỗi biên dịch**: `duplicate case 1 (constant of type int) in expression switch`. Trùng biểu thức không phải hằng thì compiler không phát hiện được.
- So sánh `case` dùng `==`, nên switch trên `any` có thể **panic runtime** nếu dynamic type hai bên giống nhau và không comparable (`comparing uncomparable type map[string]int`). Xem [operators.md](operators.md).

---

## 3. `chan`

- **Mục đích:** Khai báo kiểu **channel** — cơ chế giao tiếp và đồng bộ giữa goroutine.

**Ví dụ:**

```go
var ch chan int          // nil channel
ch = make(chan int)      // unbuffered
buf := make(chan string, 8)

ch <- 42       // gửi
v := <-ch      // nhận
close(ch)
```

**Ghi chú:**

- `chan<- T` chỉ gửi, `<-chan T` chỉ nhận; gán `chan T` sang một trong hai chiều là hợp lệ, ngược lại thì không.
- Channel `nil` block mọi thao tác gửi/nhận mãi mãi — hữu ích khi “tắt” một nhánh `select` (gán `ch = nil`).
- Gửi vào channel đã `close` → `panic: send on closed channel`; `close` hai lần → `panic: close of closed channel`. Chi tiết: [channels.md](channels.md).

---

## 4. `const`

- **Mục đích:** Khai báo hằng số biên dịch (`true`/`false`/`iota`/số/chuỗi/rune…). Hằng typed hoặc untyped.

**Ví dụ:**

```go
const Pi = 3.14159
const (
    StatusOK = 200
    StatusNotFound
)

const (
    Sunday = iota // 0
    Monday        // 1
    Tuesday       // 2
)
```

**Ghi chú:**

- Không thể lấy địa chỉ `&` của `const`. Trong khối `const`, `iota` tăng theo từng **ConstSpec** (từng dòng khai báo, kể cả dòng lặp lại biểu thức trước); reset khi gặp khối `const` mới. Dùng `_ = iota` để bỏ giá trị 0.
- Số học hằng untyped là **arbitrary precision**: spec buộc implementation biểu diễn hằng nguyên bằng ít nhất 256 bit, hằng float bằng mantissa ít nhất 256 bit. Tràn chỉ bị báo khi hằng được gán/chuyển vào một kiểu cụ thể (`constant 9223372036854775808 overflows int`).
- `len`/`cap` trên hằng chuỗi và `min`/`max` trên hằng cũng là **biểu thức hằng**:

```go
const cmin = min(3, 1, 2) // 1
const cmax = max(3, 1, 2) // 3
const clen = len("hello") // 5
```

- Chia cho hằng 0 là **lỗi biên dịch**, không phải panic: `invalid operation: division by zero`.

---

## 5. `continue`

- **Mục đích:** Bỏ phần còn lại của lần lặp hiện tại, chuyển sang lần tiếp theo; có thể kèm label.

**Ví dụ:**

```go
for i := 0; i < 10; i++ {
    if i%2 == 0 {
        continue
    }
    fmt.Println(i) // chỉ số lẻ
}
```

**Ghi chú:**

- Chỉ dùng với `for` (khác `break`, label của `continue` **bắt buộc** trỏ tới một `for` bao quanh).
- Với `range`, `continue` bỏ phần tử hiện tại nhưng vẫn chạy post-statement/bước lặp kế tiếp bình thường.
- Trong body của `for range` trên iterator function (Go 1.23+), `continue` được biên dịch thành `return true` từ `yield` — xem §19.

---

## 6. `default`

- **Mục đích:** Nhánh mặc định trong `switch` hoặc `select` (khi không có case nào sẵn sàng / khớp).

**Ví dụ:**

```go
switch n {
case 1:
    fmt.Println("one")
default:
    fmt.Println("other")
}

select {
case v := <-ch:
    fmt.Println(v)
default:
    fmt.Println("không sẵn sàng") // không block
}
```

**Ghi chú:**  
Trong `select`, `default` làm thao tác non-blocking. Chỉ được có tối đa một `default` mỗi `switch`/`select`.

---

## 7. `defer`

- **Mục đích:** Lên lịch gọi hàm **sau khi** hàm bao quanh `return` (hoặc panic), theo thứ tự LIFO.

**Ví dụ:**

```go
f, err := os.Open("a.txt")
if err != nil {
    return err
}
defer f.Close()

defer func() {
    if r := recover(); r != nil {
        fmt.Println("recovered:", r)
    }
}()
```

**Ghi chú:**

- Hàm và **đối số** của `defer` được đánh giá **ngay** tại chỗ `defer`; chỉ lời gọi bị hoãn. Biến bắt qua closure thì đọc giá trị lúc chạy.

```go
i := 0
defer fmt.Println("đối số chốt sớm:", i) // in 0
defer func() { fmt.Println("closure đọc muộn:", i) }()
i = 42
// Thứ tự in thực tế: "closure đọc muộn: 42" rồi "đối số chốt sớm: 0" (LIFO)
```

- LIFO trong **cùng một hàm**, kể cả defer đăng ký trong vòng lặp. Ba `defer` trong `for i := range 3` chạy theo thứ tự `i = 2, 1, 0`.
- `defer` gắn với **hàm**, không phải block. Đặt `defer f.Close()` trong vòng lặp dài = giữ mọi file descriptor tới khi hàm kết thúc; tách thân vòng lặp thành hàm riêng (hoặc closure gọi ngay) mới đúng.
- `recover()` chỉ dừng panic khi được gọi **trực tiếp** bởi hàm được defer. `defer func() { helper() }()` với `recover()` nằm trong `helper` sẽ trả `nil` và panic tiếp tục lan.
- `panic(nil)`: từ Go 1.21 `recover()` trả `*runtime.PanicNilError` chứ không phải `nil` (khôi phục hành vi cũ bằng `GODEBUG=panicnil=1`).
- `defer` có thể sửa **named result** — đây là cơ chế duy nhất để "chỉnh" giá trị trả về sau khi `return` đã chạy (xem §20).
- Chi phí: compiler dùng *open-coded defer* (defer inline hóa, gần như miễn phí) nhưng **bỏ** tối ưu này khi `defer` nằm trong vòng lặp, khi một hàm có hơn 8 `defer`, khi body `range`-over-func chứa `defer`, khi build với `-race`, hoặc với `-gcflags=-N`. Hot path nên tránh defer trong vòng lặp.
- Xem thêm [functions.md](functions.md), [errors.md](errors.md).

---

## 8. `else`

- **Mục đích:** Nhánh thay thế của `if`; có thể kết hợp `else if`.

**Ví dụ:**

```go
if err != nil {
    return err
} else if n == 0 {
    return errors.New("empty")
} else {
    fmt.Println("ok")
}
```

**Ghi chú:**  
Dấu `{` của `else` phải cùng dòng với `}` của `if` (quy tắc semicolon của Go). Idiom phổ biến: xử lý lỗi sớm rồi `return`, tránh `else` sâu.

---

## 9. `fallthrough`

- **Mục đích:** Buộc tiếp tục thực thi vào `case` kế tiếp trong `switch` (không kiểm tra điều kiện case đó).

**Ví dụ:**

```go
switch n {
case 1:
    fmt.Println("one")
    fallthrough
case 2:
    fmt.Println("one or two")
}
```

**Ghi chú:** bốn hạn chế đều là lỗi biên dịch, kèm thông báo thực tế của toolchain:

| Vi phạm | Thông báo compiler |
|---|---|
| Không phải statement cuối (non-empty) của case | `fallthrough statement out of place` |
| Nằm ở case cuối cùng của `switch` | `cannot fallthrough final case in switch` |
| Nằm trong type switch | `cannot fallthrough in type switch` |
| Nằm trong `select` | `fallthrough statement out of place` |

`fallthrough` chuyển điều khiển tới statement đầu của case kế tiếp **mà không đánh giá biểu thức case đó**. Ít dùng — thường thiết kế lại bằng cách liệt kê nhiều biểu thức trong một `case`.

---

## 10. `for`

- **Mục đích:** Vòng lặp duy nhất của Go: dạng 3 mệnh đề, while-style, infinite, hoặc `for range`.

**Ví dụ:**

```go
for i := 0; i < 10; i++ { /* ... */ }

for n < 100 { // while-style
    n *= 2
}

for { // infinite
    break
}

for i, v := range slice {
    fmt.Println(i, v)
}
```

**Ghi chú:**

- Không có `while`/`do-while` riêng; không có `do {} while`.
- Từ Go 1.22, **mọi** dạng `for` khai báo biến bằng `:=` (cả dạng 3 mệnh đề và `for range`) tạo biến **mới mỗi lần lặp** — closure/goroutine bắt đúng giá trị.
- `range` nhận thêm integer (Go 1.22) và iterator function (Go 1.23) — xem §19.
- Chi tiết mọi dạng và bẫy: [statements.md](statements.md).

---

## 11. `func`

- **Mục đích:** Khai báo hàm, method, literal hàm (closure), hoặc kiểu hàm.

**Ví dụ:**

```go
func Add(a, b int) int {
    return a + b
}

type Handler func(http.ResponseWriter, *http.Request)

fn := func(x int) int { return x * 2 }
```

**Ghi chú:**  
Go không có overload theo chữ ký. Method gắn với kiểu qua receiver: `func (t *T) M()`. Xem [functions.md](functions.md) và [methods-interfaces.md](methods-interfaces.md).

---

## 12. `go`

- **Mục đích:** Khởi chạy một **goroutine** — đơn vị đồng thời nhẹ của Go.

**Ví dụ:**

```go
go func() {
    fmt.Println("chạy nền")
}()

go process(ctx, job)
```

**Ghi chú:**

- Hàm và **đối số** của lời gọi được đánh giá trong goroutine gọi, tại chỗ viết `go`; chỉ thân hàm chạy ở goroutine mới.
- Không có giá trị trả về trực tiếp từ `go`; dùng channel, `sync.WaitGroup`, `errgroup`, hoặc context. Leak goroutine là lỗi phổ biến — luôn có điều kiện thoát.
- Từ Go 1.22, biến vòng lặp là per-iteration nên `go func() { use(v) }()` trong `for range` đã đúng, không cần `v := v`. `go fix` có modernizer `forvar` để xóa các dòng `v := v` dư thừa.
- Từ Go 1.25 có `(*sync.WaitGroup).Go(f func())` gộp `Add(1)` + `go` + `Done()`:

```go
var wg sync.WaitGroup
for _, job := range jobs {
    wg.Go(func() { process(job) }) // Go 1.25+
}
wg.Wait()
```

- `go fix` có modernizer `waitgroup` tự chuyển mẫu `wg.Add(1)/go/wg.Done()` sang `wg.Go`. Chi tiết đồng thời: [concurrency.md](concurrency.md).

---

## 13. `goto`

- **Mục đích:** Nhảy tới một **label** trong cùng hàm.

**Ví dụ:**

```go
func parse() error {
    // ...
    if err != nil {
        goto Fail
    }
    return nil
Fail:
    cleanup()
    return err
}
```

**Ghi chú:** hai luật của spec, kèm thông báo compiler thực tế:

| Luật | Ví dụ sai | Thông báo |
|---|---|---|
| Không được làm biến mới "vào scope" (nhảy qua khai báo) | `goto L` rồi `v := 3` rồi `L:` | `goto L jumps over declaration of v` |
| `goto` bên ngoài một block không nhảy được vào trong block đó | `goto L1` rồi `for { L1: }` | `goto L1 jumps into block starting at ...` |

- Label thuộc phạm vi **toàn thân hàm** (không phải block), nhưng không xuyên qua biên hàm — không `goto` từ closure ra ngoài.
- Label khai báo mà không dùng là lỗi biên dịch (`label L defined and not used`).
- Hiếm khi cần; ưu tiên `return`/`break`/`defer`. Hữu ích trong code sinh tự động, parser trạng thái, hoặc đường thoát cleanup tuyến tính.

---

## 14. `if`

- **Mục đích:** Rẽ nhánh có điều kiện; hỗ trợ câu lệnh khởi tạo trước điều kiện.

**Ví dụ:**

```go
if n > 0 {
    fmt.Println("positive")
}

if v, ok := m[key]; ok {
    fmt.Println(v)
}
```

**Ghi chú:**  
Điều kiện phải là `bool` (không ép từ số như C). Biến khai báo trong init chỉ sống trong khối `if`/`else`.

---

## 15. `import`

- **Mục đích:** Import package khác vào file hiện tại.

**Ví dụ:**

```go
import "fmt"

import (
    "io"
    "os"

    f "fmt"           // alias
    _ "image/png"     // side-effect import (init)
    . "math"          // dot import (tránh trong production)
)
```

**Ghi chú:**  
Import không dùng → lỗi biên dịch. `_` chỉ để chạy `init` của package. Đường dẫn module theo `go.mod`.

---

## 16. `interface`

- **Mục đích:** Định nghĩa kiểu interface (tập method) hoặc kiểu cụ thể `interface{...}` / `any`.

**Ví dụ:**

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

var x any = 42 // tương đương interface{}
```

**Ghi chú:**

- Thỏa mãn interface là **ngầm** (không cần `implements`). Go 1.18+ dùng interface làm constraint trong generic: `comparable`, `~int | ~string`, union.
- Interface có type set (union, `~T`) chỉ dùng được làm **constraint**, không dùng làm kiểu biến.
- Từ Go 1.26, một generic type được phép **tự tham chiếu** trong danh sách type parameter của chính nó — hữu ích cho constraint kiểu "self":

```go
type Adder[A Adder[A]] interface {
    Add(A) A
}
```

- Chi tiết: [methods-interfaces.md](methods-interfaces.md), [generics.md](generics.md), [typesystem.md](typesystem.md).

---

## 17. `map`

- **Mục đích:** Kiểu map (ánh xạ key → value); key phải comparable.

**Ví dụ:**

```go
var m map[string]int           // nil map
m = make(map[string]int)
m["a"] = 1

v, ok := m["a"] // comma-ok
delete(m, "a")
```

**Ghi chú:**

- Đọc từ `nil` map cho zero value, `len` = 0, `range` = 0 lần lặp; ghi vào `nil` map → `panic: assignment to entry in nil map`.
- Key phải comparable **lúc runtime**: `map[any]V` với key là slice/map/func → `panic: runtime error: hash of unhashable type []int`.
- `NaN` làm key thì chèn được nhưng **không tra lại được** (`m[NaN]` luôn cho zero value); `clear(m)` vẫn xóa được các entry đó.
- Thứ tự `range` map là **ngẫu nhiên hóa có chủ đích**; cần ổn định thì `slices.Sorted(maps.Keys(m))` (Go 1.23+).
- Map không an toàn giữa các goroutine — dùng mutex hoặc `sync.Map`. Chi tiết: [collections.md](collections.md).

---

## 18. `package`

- **Mục đích:** Khai báo tên package của file; dòng đầu tiên (không kể comment) của mỗi file `.go`.

**Ví dụ:**

```go
package main

package http
```

**Ghi chú:**  
Mọi file trong cùng thư mục phải cùng tên package (trừ `_test` với `package x_test` cho external test). `main` + `func main()` tạo executable.

---

## 19. `range`

- **Mục đích:** Lặp qua array, pointer-to-array, slice, string, map, channel, **số nguyên** (Go 1.22, tính năng ổn định trong spec — không phải thí nghiệm) và **iterator function** (Go 1.23).

**Ví dụ:**

```go
for i, v := range []int{10, 20} {
    fmt.Println(i, v)
}

for k, v := range m {
    fmt.Println(k, v)
}

for v := range ch { // nhận đến khi channel đóng
    fmt.Println(v)
}

// Go 1.22+: lặp i = 0..n-1 trên mọi kiểu integer
for i := range 5 {
    fmt.Println(i)
}

// Go 1.23+: range over func (iterator)
for k, v := range maps.All(m) {
    fmt.Println(k, v)
}
```

**Ghi chú:** bảng số biến nhận được và ngữ nghĩa:

| Toán hạng `range x` | Biến thứ nhất | Biến thứ hai |
|---|---|---|
| `[n]T`, `*[n]T`, `[]T` | index `int` | phần tử |
| `string` | byte offset của rune | `rune` |
| `map[K]V` | key | value |
| `chan T` | phần tử nhận được | — (không có) |
| integer `n` | `0..n-1`, cùng kiểu với `n` | — (không có) |
| `func(func() bool)` | — | — |
| `func(func(V) bool)` = `iter.Seq[V]` | value do `yield` đẩy ra | — |
| `func(func(K, V) bool)` = `iter.Seq2[K,V]` | value thứ nhất | value thứ hai |

Chi tiết dễ sai:

- **String:** duyệt Unicode code point (rune), không phải byte; index nhảy theo độ rộng UTF-8. Byte không hợp lệ cho ra `U+FFFD` với độ rộng 1 (`"a\xffb"` → `0:'a' 1:'\uFFFD' 2:'b'`). Duyệt byte thì dùng `for i := 0; i < len(s); i++`.
- **Integer:** `n <= 0` cho **0 lần lặp** (không panic). `n` có thể là `int8`, `uint`, hằng untyped… và biến lặp mang đúng kiểu đó.
- **Array (không phải pointer):** toán hạng được **copy** một lần trước vòng lặp, nên sửa `a[i]` trong thân không ảnh hưởng giá trị `v` đọc ra. Với `*[n]T` thì không copy.
- **Slice:** `len` được đánh giá **một lần** trước vòng lặp — `append` trong thân không làm vòng lặp dài thêm.
- **`nil` slice / `nil` map:** 0 lần lặp. **`nil` channel:** block mãi. **`nil` iterator function:** panic nil pointer dereference.
- Biến vòng lặp được tạo lại **mỗi lần lặp** từ Go 1.22 (tránh bug closure kinh điển).
- Với iterator: `break`/`return`/`goto` ra ngoài làm `yield` trả `false`; nếu iterator vẫn gọi `yield` sau đó thì panic `range function continued iteration after function for loop body returned false`.

Xem thêm [statements.md](statements.md), [collections.md](collections.md).

---

## 20. `return`

- **Mục đích:** Trả về từ hàm; có thể nhiều giá trị; với named result có thể `return` trống (naked return).

**Ví dụ:**

```go
func Div(a, b int) (int, error) {
    if b == 0 {
        return 0, errors.New("division by zero")
    }
    return a / b, nil
}

func Sum(xs []int) (total int) {
    for _, x := range xs {
        total += x
    }
    return // naked return
}
```

**Ghi chú:**

- Naked return chỉ nên dùng ở hàm ngắn (`gopls`/reviewer thường phàn nàn ở hàm dài).
- `return expr` với named result là **hai bước**: gán `expr` vào biến result, rồi chạy các `defer`, rồi mới trả về caller. Vì vậy `defer` đọc/ghi được named result:

```go
func Do() (err error) {
    defer func() {
        if err != nil {
            err = fmt.Errorf("do: %w", err) // caller thấy lỗi đã bọc
        }
    }()
    return doWork()
}
```

- Với result **không** đặt tên, `defer` không thể thay đổi giá trị trả về (trừ khi panic + recover, lúc đó result giữ zero value).
- `return` trong body của `for range` trên iterator function thoát khỏi cả hàm bao ngoài (compiler lo phần `yield` trả `false`).

---

## 21. `select`

- **Mục đích:** Chờ đồng thời nhiều thao tác channel; chọn một case sẵn sàng (ngẫu nhiên nếu nhiều).

**Ví dụ:**

```go
select {
case msg := <-in:
    fmt.Println(msg)
case <-ctx.Done():
    return ctx.Err()
case out <- result:
    // đã gửi
}
```

**Ghi chú:**

- Mọi biểu thức channel và mọi giá trị cần gửi trong **tất cả** các case được đánh giá **một lần, theo thứ tự nguồn**, trước khi `select` chọn. Nghĩa là `case out <- expensive():` gọi `expensive()` dù case đó không được chọn.
- Không case nào sẵn sàng và không có `default` → block. Nếu **mọi** channel là `nil` (hoặc `select {}`) thì goroutine block mãi; chương trình đơn goroutine sẽ chết với `fatal error: all goroutines are asleep - deadlock!`.
- Nhiều case sẵn sàng → chọn **giả ngẫu nhiên đều**, không theo thứ tự viết. Đo thực tế 1000 vòng với hai channel luôn sẵn sàng cho tỉ lệ xấp xỉ 50/50 (một lần chạy: 536 / 464).
- Gán `ch = nil` để **vô hiệu hóa** một nhánh `select` vĩnh viễn — idiom chuẩn cho fan-in khi từng nguồn lần lượt đóng.
- `default` biến `select` thành non-blocking; đặt trong `for` mà không có gì chặn lại thì thành busy-loop ăn CPU.
- Kết hợp `time.After` / `context` để timeout. Lời khuyên cũ "tránh `time.After` vì timer không được GC" đã hết hiệu lực: doc của `time.After` ghi rõ từ Go 1.23 GC thu hồi được timer chưa `Stop`, "There is no reason to prefer `NewTimer` when `After` will do". Xem [statements.md](statements.md), [channels.md](channels.md).

---

## 22. `struct`

- **Mục đích:** Định nghĩa kiểu cấu trúc (tập field); hỗ trợ embedding.

**Ví dụ:**

```go
type Point struct {
    X, Y float64
}

type Circle struct {
    Point  // embedding
    Radius float64
}

p := Point{X: 1, Y: 2}
```

**Ghi chú:**  
Field xuất khẩu bắt đầu bằng chữ hoa. Struct so sánh được nếu mọi field comparable. Zero value có field = zero tương ứng.

---

## 23. `switch`

- **Mục đích:** Rẽ nhánh nhiều nhánh; expression switch hoặc type switch (`switch x.(type)`).

**Ví dụ:**

```go
switch {
case n < 0:
    fmt.Println("neg")
case n == 0:
    fmt.Println("zero")
default:
    fmt.Println("pos")
}

switch v := any(x).(type) {
case int:
    fmt.Println("int", v)
case string:
    fmt.Println("string", v)
}
```

**Ghi chú:**

- Không cần `break` (không fall-through mặc định). `switch` không biểu thức tương đương `switch true`.
- `switch` có init statement: `switch x := f(); x { ... }`; hoặc chỉ init: `switch n := f(); { case n < 0: ... }`.
- `default` không bắt buộc ở cuối, nhưng đặt cuối là quy ước.
- Không có `fallthrough` trong type switch; `case` liệt kê nhiều kiểu thì biến giữ kiểu interface gốc.
- Chi tiết cùng type switch: [statements.md](statements.md), [methods-interfaces.md](methods-interfaces.md).

---

## 24. `type`

- **Mục đích:** Khai báo tên kiểu mới, alias (`type A = B`), hoặc type switch / assertion liên quan cú pháp.

**Ví dụ:**

```go
type ID int

type Handler func(ctx context.Context) error

type User struct {
    Name string
}

// Go 1.9+: alias
type Vector = []float64

// Go 1.24+: alias có type parameter
type Pair[K comparable, V any] = map[K][]V
```

**Ghi chú:**

- `type ID int` tạo **defined type** mới: không gán ngầm với `int`, không kế thừa method, nhưng dùng chung tập toán tử của underlying type.
- `type Vector = []float64` chỉ là tên khác cho **cùng** kiểu — mọi method set, so sánh, gán đều giống hệt.
- Alias **có type parameter** hợp lệ từ Go 1.24 (`type Set[T comparable] = map[T]struct{}`).
- Trong hàm, `type` tạo kiểu local; kiểu local vẫn có thể có method thông qua embedding, nhưng không khai báo method trực tiếp được.
- Chi tiết defined vs alias, comparable, conversion: [typesystem.md](typesystem.md).

---

## 25. `var`

- **Mục đích:** Khai báo biến (package-level hoặc local); có thể dùng khối `var (...)`.

**Ví dụ:**

```go
var count int
var name = "Go"
var (
    a int
    b string = "hi"
)

// tương đương short declaration trong hàm:
// count := 0
```

**Ghi chú:**  
Biến luôn được khởi tạo zero value nếu không gán. `:=` chỉ trong hàm. Package-level chỉ dùng `var`/`const`/`type`/`func`, không dùng `:=`.

---

## 26. Predeclared identifiers (không phải keyword)

Các tên sau **không phải** từ khóa — chúng là identifier khai báo ngầm trong **universe block**, nên có thể shadow (không nên) và có thể dùng làm tên field/method. Đây là lý do `min`, `max`, `clear`, `new`, `make`, `any`, `nil` không nằm trong danh sách 25 keyword và việc thêm chúng không phá vỡ code cũ:

### Hằng & zero đặc biệt

| Identifier | Ý nghĩa |
|------------|---------|
| `true`, `false` | Hằng `bool` |
| `nil` | Zero của pointer, slice, map, chan, func, interface |
| `iota` | Bộ đếm trong khối `const` |

### Kiểu predeclared

`any` (= `interface{}`), `bool`, `byte` (= `uint8`), `comparable`, `complex64/128`, `error`, `float32/64`, `int`/`int8`…`int64`, `rune` (= `int32`), `string`, `uint`…`uintptr`.

### Hàm built-in (đủ 18, theo spec)

| Hàm | Vai trò ngắn | Từ |
|-----|----------------|----|
| `append` | Thêm phần tử vào slice | 1.0 |
| `cap` | Capacity (slice/array/`*array`/chan) | 1.0 |
| `clear` | Xóa toàn bộ map / zero mọi phần tử slice (len không đổi) | 1.21 |
| `close` | Đóng channel | 1.0 |
| `complex` / `real` / `imag` | Số phức | 1.0 |
| `copy` | Sao chép slice (và `string` → `[]byte`) | 1.0 |
| `delete` | Xóa key khỏi map | 1.0 |
| `len` | Độ dài | 1.0 |
| `make` | Tạo slice/map/chan (chỉ ba kiểu này) | 1.0 |
| `max` / `min` | Lớn nhất / nhỏ nhất của ≥ 1 toán hạng ordered | 1.21 |
| `new` | Cấp phát; `new(T)` cho zero value, `new(expr)` cho bản copy của `expr` | 1.0 / expr: 1.26 |
| `panic` / `recover` | Dừng / bắt panic | 1.0 |
| `print` / `println` | Debug bootstrap (tránh production) | 1.0 |

Go 1.26 mở rộng `new` để nhận **biểu thức**, không chỉ kiểu — thay thế mẫu "biến tạm rồi lấy địa chỉ":

```go
p := new(42)      // *int trỏ tới 42
b := new(true)    // *bool
s := new("hi")    // *string

x := S{A: 7}
q := new(x)       // *S chứa BẢN COPY của x tại thời điểm gọi
x.A = 9           // q.A vẫn là 7
```

- Hằng untyped được chuyển sang default type trước: `new(42)` là `*int`, `new(4.2)` là `*float64`.
- `new(nil)` là lỗi: `use of untyped nil in argument to new`.
- Modernizer `newexpr` của `go fix` tự viết lại mẫu cũ sang `new(expr)`.

**Ghi chú:**

- Built-in không import được và **không phải giá trị**: không gán `f := len`, không truyền `len` làm đối số.
- `min`/`max` với `NaN` trả về `NaN`; `min(-0.0, 0.0)` trả về `-0.0` (kiểm chứng bằng `math.Signbit`).
- Shadow `len`/`make`/`min` bằng biến cùng tên sẽ che built-in trong scope đó — tránh tuyệt đối.

---

## 27. Ngữ nghĩa keyword theo phiên bản & modernizer

Danh sách 25 keyword không đổi, nhưng **hành vi** của một số keyword đã thay đổi:


| Version | Keyword bị ảnh hưởng | Thay đổi |
| ------- | -------------------- | -------- |
| 1.9  | `type` | alias `type A = B` |
| 1.13 | `<<` `>>` (toán tử, xem [operators.md](operators.md)) | shift count được phép là signed integer |
| 1.18 | `interface`, `type`, `func` | generics: type parameter, interface làm constraint |
| 1.21 | (predeclared) | thêm built-in `min`, `max`, `clear`; `panic(nil)` → `*runtime.PanicNilError` |
| 1.22 | `for`, `range` | `for i := range n` trên integer; biến vòng lặp per-iteration |
| 1.23 | `for`, `range` | range over func (`iter.Seq`, `iter.Seq2`) |
| 1.24 | `type` | generic type alias |
| 1.25 | — | không có thay đổi ngôn ngữ; spec bỏ khái niệm "core type" |
| 1.26 | (predeclared), `interface` | `new(expr)`; generic type tự tham chiếu trong type parameter list |

Từ Go 1.26, `go fix` được viết lại trên nền vet analysis framework và trở thành nơi chứa các **modernizer** — chạy `go fix ./...` (hoặc `go fix -diff ./...` để xem patch) để tự động hiện đại hóa code. Xem danh sách bằng `go tool fix help`. Những modernizer liên quan trực tiếp tới keyword trong file này:


| Modernizer | Việc nó làm |
| ---------- | ----------- |
| `rangeint` | vòng `for` 3 mệnh đề → `for i := range n` |
| `forvar` | bỏ dòng `v := v` dư thừa trong vòng lặp (không cần từ 1.22) |
| `newexpr` | dùng `new(expr)` của Go 1.26 |
| `minmax` | `if`/`else` chọn lớn/nhỏ → `min` / `max` |
| `waitgroup` | `wg.Add(1)` + `go` + `wg.Done()` → `wg.Go` |
| `any` | `interface{}` → `any` |
| `stringsseq` | `range` trên `strings.Split` → `strings.SplitSeq` |
| `stditerators` | API kiểu `Len`/`At` → iterator |

---

## Tóm tắt nhanh

- Đúng **25 keywords**, không đổi từ Go 1 — mọi thứ khác (kể cả `int`, `nil`, `make`, `min`, `any`) chỉ là predeclared identifier trong universe block.
- Keyword không thể dùng làm identifier; predeclared identifier thì được (nhưng đừng).
- Ba thay đổi hành vi dễ bị viết sai trong tài liệu cũ: `for i := range n` **là** tính năng ổn định (1.22), biến vòng lặp **là** per-iteration (1.22), `new` **nhận biểu thức** (1.26).
- Học sâu hành vi: [statements.md](statements.md), [operators.md](operators.md), [functions.md](functions.md), [methods-interfaces.md](methods-interfaces.md), [typesystem.md](typesystem.md).
