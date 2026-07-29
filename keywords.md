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

Go có đúng **25 từ khóa dành riêng (reserved keywords)**. Không dùng chúng làm tên biến, hàm, kiểu, hay package.

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
`break` không có nhãn chỉ ảnh hưởng vòng/`switch`/`select` bao quanh gần nhất. Dùng label khi cần thoát nhiều tầng.

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
Một `case` có thể liệt kê nhiều biểu thức (cách nhau bằng dấu phẩy). Trong `select`, mỗi `case` là một thao tác channel.

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
`chan<- T` chỉ gửi, `<-chan T` chỉ nhận. Channel `nil` sẽ block mọi thao tác gửi/nhận mãi mãi — hữu ích khi “tắt” một nhánh `select`.

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
Không thể lấy địa chỉ `&` của `const`. Trong khối `const`, `iota` tăng theo từng dòng; reset khi gặp khối `const` mới.

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
Chỉ dùng với `for`. Với `range`, `continue` bỏ phần tử hiện tại.

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
Đối số của `defer` được đánh giá **ngay** tại chỗ `defer`, nhưng lời gọi chạy muộn. Dùng cho unlock, đóng file, cleanup. Xem thêm [functions.md](functions.md).

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

**Ghi chú:**  
Phải là **câu lệnh cuối** của `case`. Không dùng trong type switch. Ít dùng — thường thiết kế lại bằng nhiều biểu thức trong một `case`.

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
Không có `while`/`do-while` riêng. Chi tiết xem [statements.md](statements.md).

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
Không có giá trị trả về trực tiếp từ `go`; dùng channel, `sync.WaitGroup`, hoặc context để đồng bộ/kết quả. Leak goroutine là lỗi phổ biến — luôn có điều kiện thoát.

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

**Ghi chú:**  
Không được nhảy vào scope biến mới (qua khai báo). Hiếm khi cần; ưu tiên `return`/`break`/`defer`. Hữu ích trong code sinh tự động hoặc tối ưu đặc biệt.

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
Thỏa mãn interface là **ngầm** (không cần `implements`). Go 1.18+ dùng constraint trong generic: `comparable`, `~int`, v.v. Chi tiết: [methods-interfaces.md](methods-interfaces.md).

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
Đọc từ `nil` map cho zero value; ghi vào `nil` map → panic. Map không an toàn giữa các goroutine — dùng `sync.Map` hoặc mutex.

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

- **Mục đích:** Lặp qua array, slice, string, map, channel (và từ Go 1.22+ có thể dùng trong `for` integer trong một số ngữ cảnh thí nghiệm — chuẩn phổ biến vẫn là các kiểu trên; Go 1.22: `for i := range n`).

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

// Go 1.22+: lặp 0..n-1
for i := range 5 {
    fmt.Println(i)
}
```

**Ghi chú:**  
Với string, `range` duyệt Unicode code point (rune), không phải byte. Biến vòng lặp được tạo lại mỗi lần lặp từ Go 1.22 (tránh bug closure cũ).

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
Naked return chỉ nên dùng hàm ngắn. `defer` chạy trước khi giá trị thực sự trả về caller.

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
Không có case sẵn sàng và không có `default` → block. Kết hợp `time.After` / context để timeout. Xem [statements.md](statements.md).

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
Không cần `break` (không fall-through mặc định). `switch` không biểu thức tương đương `switch true`.

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
```

**Ghi chú:**  
`type ID int` tạo kiểu **mới** (cần chuyển đổi tường minh với `int`). `type Vector = []float64` chỉ là tên khác, cùng kiểu.

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

Các tên sau **không phải** từ khóa — có thể shadow (không nên), và thuộc universe block:

### Hằng & zero đặc biệt

| Identifier | Ý nghĩa |
|------------|---------|
| `true`, `false` | Hằng `bool` |
| `nil` | Zero của pointer, slice, map, chan, func, interface |
| `iota` | Bộ đếm trong khối `const` |

### Kiểu predeclared

`any` (= `interface{}`), `bool`, `byte` (= `uint8`), `comparable`, `complex64/128`, `error`, `float32/64`, `int`/`int8`…`int64`, `rune` (= `int32`), `string`, `uint`…`uintptr`.

### Hàm built-in (một phần)

| Hàm | Vai trò ngắn |
|-----|----------------|
| `append` | Thêm phần tử vào slice |
| `cap` | Capacity (slice/array/chan) |
| `clear` | Xóa map / zero slice (Go 1.21+) |
| `close` | Đóng channel |
| `complex` / `real` / `imag` | Số phức |
| `copy` | Sao chép slice |
| `delete` | Xóa key khỏi map |
| `len` | Độ dài |
| `make` | Tạo slice/map/chan |
| `max` / `min` | So sánh (Go 1.21+) |
| `new` | Cấp phát, trả `*T` (zero) |
| `panic` / `recover` | Dừng / bắt panic |
| `print` / `println` | Debug bootstrap (tránh production) |

**Ghi chú:**  
Built-in không import được; gọi trực tiếp. Shadow `len`/`make` bằng biến cùng tên sẽ che built-in trong scope đó — tránh tuyệt đối.

---

## Tóm tắt nhanh

- Đúng **25 keywords** — mọi thứ khác (kể cả `int`, `nil`, `make`) chỉ là predeclared.  
- Keyword không thể dùng làm identifier.  
- Học sâu hành vi: [statements.md](statements.md), [functions.md](functions.md), [methods-interfaces.md](methods-interfaces.md).
