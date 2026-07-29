# Phát biểu (Statements) trong Go

---

## Mục lục

1. [Tổng quan](#1-tổng-quan)
2. [Khối, phạm vi & shadowing](#2-khối-phạm-vi--shadowing)
3. [Khai báo biến](#3-khai-báo-biến)
4. [`if` / `else`](#4-if--else)
5. [`switch` & type switch](#5-switch--type-switch)
6. [`for` — mọi dạng vòng lặp](#6-for--mọi-dạng-vòng-lặp)
7. [`break`, `continue` & label](#7-break-continue--label)
8. [`goto`](#8-goto)
9. [`defer`](#9-defer)
10. [`select` (tóm tắt)](#10-select-tóm-tắt)
11. [Empty statement](#11-empty-statement)
12. [Mẹo & best practices](#12-mẹo--best-practices)
13. [Bẫy statement khi review code](#13-bẫy-statement-khi-review-code)
14. [Tính năng theo phiên bản](#14-tính-năng-theo-phiên-bản)

---

## 1. Tổng quan

Trong Go, **statement** là đơn vị điều khiển luồng và khai báo. Khác expression (cho giá trị), statement thực thi hành động.

Nhóm chính:

- **Declaration:** `var`, `const`, `type`, short `:=`
- **Assignment / expression statement:** gán, gọi hàm, `go`, `defer`, `++`/`--`
- **Control:** `if`, `switch`, `for`, `select`
- **Jump:** `break`, `continue`, `goto`, `return`, `fallthrough`
- **Block:** `{ ... }` tạo scope

Go **tự chèn semicolon** theo quy tắc lexing — vì vậy `{` thường phải cùng dòng với `if`/`for`/`func`.

---

## 2. Khối, phạm vi & shadowing

### 2.1 Block và scope

```go
package main

var global = 1 // package block

func main() {
    x := 10 // function block
    {
        y := 20 // inner block
        fmt.Println(x, y)
    }
    // y không còn visible
}
```

Thứ tự scope (từ ngoài vào trong): **universe** → **package** → **file** (imports) → **function** → **block**.

### 2.2 Shadowing

Khai báo tên trùng ở scope trong **che** tên ngoài:

```go
err := doA()
if err != nil {
    return err
}

if err := doB(); err != nil { // err mới, shadow err ngoài
    return err
}
// err ngoài vẫn là kết quả doA
```

- Short `:=` chỉ hợp lệ khi **ít nhất một tên bên trái là mới trong block hiện tại**; các tên đã tồn tại chỉ được **gán lại** (không tạo biến mới). Nếu mọi tên đã tồn tại trong cùng block → lỗi biên dịch `no new variables on left side of :=`.
- Điểm quan trọng: "block hiện tại", không phải "hàm hiện tại". `if err := f(); ...` luôn tạo `err` **mới** vì thân `if` là block khác — đây chính là nguồn của bug shadowing.
- Tránh shadow `err`, `ctx`, `ok` một cách khó theo dõi. `go vet` **không** có analyzer shadow bật mặc định; dùng `gopls`/linter riêng nếu cần.

```go
x := 1
x, y := 2, 3 // OK: y mới trong block này, x chỉ được gán lại
// x, y := 4, 5 // LỖI: no new variables on left side of :=
x, y = 4, 5  // assignment thuần
```

Bẫy kinh điển: `:=` trong block con tạo biến mới nên kết quả gán bị mất khi ra khỏi block.

```go
func load(path string) (*Config, error) {
    var cfg *Config
    if raw, err := os.ReadFile(path); err == nil {
        cfg, err := parse(raw) // BUG: cfg VÀ err đều là biến mới của block này
        _, _ = cfg, err        // cfg ngoài vẫn nil
    }
    return cfg, nil // luôn trả nil, không lỗi
}
```

Sửa bằng cách tách khai báo khỏi gán:

```go
func load(path string) (*Config, error) {
    raw, err := os.ReadFile(path)
    if err != nil {
        return nil, err
    }
    cfg, err := parse(raw) // cùng block với raw/err → err được gán lại
    if err != nil {
        return nil, err
    }
    return cfg, nil
}
```

---

## 3. Khai báo biến

### 3.1 `var`

```go
var a int              // 0
var b = 42             // suy luận int
var c, d string = "x", "y"
var (
    host = "localhost"
    port = 8080
)
```

Package-level phải dùng `var` (không `:=`).

### 3.2 Short declaration `:=`

Chỉ **trong hàm**:

```go
sum := 0
name, err := os.Hostname()
```

- Ít nhất một biến bên trái phải **mới** trong block hiện tại.
- Kiểu suy luận từ biểu thức bên phải.

### 3.3 Zero value

Mọi biến có giá trị zero hữu ích: `0`, `false`, `""`, `nil` (pointer/slice/map/chan/func/interface).

### 3.4 `const` / `type` trong statement list

Ở cấp package hoặc trong hàm:

```go
func f() {
    const max = 100
    type localID int
    var id localID = 1
    _ = id
}
```

---

## 4. `if` / `else`

### 4.1 Cơ bản

```go
if n > 0 {
    fmt.Println("positive")
} else if n == 0 {
    fmt.Println("zero")
} else {
    fmt.Println("negative")
}
```

Điều kiện **phải** là `bool`. Không viết `if n { }` như C.

### 4.2 Init statement

```go
if v, ok := m[key]; ok {
    fmt.Println(v)
} else {
    // v, ok vẫn visible trong else
    fmt.Println("missing")
}
// v, ok không còn ở đây
```

Idiom xử lý lỗi:

```go
if err := work(); err != nil {
    return fmt.Errorf("work: %w", err)
}
```

### 4.3 Không có ternary

Dùng `if`/`else` hoặc map/function nhỏ — không có `cond ? a : b`.

---

## 5. `switch` & type switch

### 5.1 Expression switch

```go
switch day {
case "Mon", "Tue", "Wed", "Thu", "Fri":
    fmt.Println("weekday")
case "Sat", "Sun":
    fmt.Println("weekend")
default:
    fmt.Println("unknown")
}
```

- Không fall-through mặc định; muốn thì `fallthrough` (phải là statement non-empty **cuối** của case, không được ở case cuối cùng, không dùng trong type switch/`select`).
- `case` là biểu thức, có thể gọi hàm. Các case được đánh giá và so sánh **theo thứ tự viết**, từ trên xuống, và chỉ đến khi khớp.
- So sánh dùng `==`, nên giá trị switch phải comparable với các case. Trên `any` việc này có thể **panic lúc chạy**: `switch x { case y: }` với `x`, `y` đều là `map[string]int` → `panic: runtime error: comparing uncomparable type map[string]int`. Nếu dynamic type khác nhau thì chỉ ra `false`, không panic.
- Trùng **hằng** trong cùng switch là lỗi biên dịch: `duplicate case 1 (constant of type int) in expression switch`.
- `case` rỗng không làm gì (không rơi xuống case dưới) — dùng để "bỏ qua có chủ ý".

### 5.2 `switch` không biểu thức (= `switch true`)

```go
switch {
case score >= 90:
    fmt.Println("A")
case score >= 80:
    fmt.Println("B")
default:
    fmt.Println("C or below")
}
```

### 5.3 Init trong switch

```go
switch n := compute(); {
case n < 0:
    fmt.Println("neg")
case n > 0:
    fmt.Println("pos")
default:
    fmt.Println("zero")
}
```

### 5.4 Type switch

```go
func describe(x any) {
    switch v := x.(type) {
    case nil:
        fmt.Println("nil")
    case int:
        fmt.Println("int", v)
    case string:
        fmt.Println("string", v)
    case []byte:
        fmt.Println("bytes", len(v))
    default:
        fmt.Printf("other %T\n", v)
    }
}
```

- `v` mang kiểu của từng `case`.
- Không dùng `fallthrough` trong type switch (`cannot fallthrough in type switch`).
- Case có thể liệt kê nhiều kiểu: `case int, int64:` → `v` giữ kiểu **interface tĩnh** của biểu thức switch (ở đây là `any`), không phải kiểu cụ thể.
- `case nil` khớp khi interface **rỗng cả type lẫn value**. Một interface chứa typed nil (`(*T)(nil)`) **không** khớp `case nil` mà khớp `case *T` — xem typed nil trong [typesystem.md](typesystem.md).
- `switch v := x.(type)` mà không dùng `v` ở bất kỳ case nào là lỗi `v declared and not used`; viết `switch x.(type)` nếu chỉ cần kiểu.
- Với `error`, đừng type-switch trực tiếp trên `err` — dùng `errors.As` (hoặc `errors.AsType[E]` từ Go 1.26) để đi qua chuỗi wrap. Xem [errors.md](errors.md).

---

## 6. `for` — mọi dạng vòng lặp

Go chỉ có `for`.

### 6.1 Ba mệnh đề (C-style)

```go
for i := 0; i < 10; i++ {
    fmt.Println(i)
}
```

Mệnh đề có thể trống một phần: `for ; cond; post {}`.

### 6.2 While-style

```go
for n < 100 {
    n *= 2
}
```

### 6.3 Infinite

```go
for {
    if done() {
        break
    }
    work()
}
```

### 6.4 `for range`

```go
nums := []int{10, 20, 30}

for i, v := range nums {
    fmt.Println(i, v)
}

for _, v := range nums { // bỏ index
    fmt.Println(v)
}

for i := range nums { // chỉ index
    fmt.Println(i)
}

// string → rune (không phải byte)
for i, r := range "Go語言" {
    fmt.Printf("%d: %c\n", i, r)
}

// map — thứ tự không đảm bảo
for k, v := range m {
    fmt.Println(k, v)
}

// channel — đến khi đóng
for v := range ch {
    handle(v)
}
```

Ngữ nghĩa đánh giá — phần hay bị hiểu sai:

| Toán hạng | Được đánh giá / copy như thế nào |
|---|---|
| `[]T` | biểu thức slice đánh giá **một lần**; `len` chốt trước vòng lặp → `append` trong thân không kéo dài vòng lặp |
| `[n]T` | **copy toàn bộ array** một lần → sửa `a[i]` trong thân không đổi `v` đọc ra |
| `*[n]T` | **không** copy; sửa qua pointer thấy ngay ở các lần lặp sau. `range` trên `*[n]T` **nil** vẫn chạy `n` lần nếu chỉ dùng index (độ dài là hằng của kiểu) |
| `map[K]V` | thứ tự ngẫu nhiên; entry thêm trong lúc lặp *có thể* được duyệt hoặc không; entry đã `delete` sẽ không được duyệt |
| `string` | duyệt rune; byte không hợp lệ → `U+FFFD` rộng 1 byte |
| `chan T` | lặp tới khi channel `close`; `nil` channel → block mãi |
| integer | đánh giá một lần; `n <= 0` → 0 lần lặp |
| iterator function | gọi iterator đúng một lần, truyền vào `yield` do compiler sinh |

### 6.5 `for range` trên integer (Go 1.22+)

Tính năng **ổn định** của spec từ Go 1.22, không phải thí nghiệm. Trên toolchain cũ hơn, compiler báo `cannot range over 3 (untyped int constant): requires go1.22 or later`.

```go
for i := range 5 {
    fmt.Println(i) // 0,1,2,3,4
}

for range 3 { // không cần biến: lặp đúng 3 lần
    tick()
}

var n uint8 = 3
for i := range n { // i có kiểu uint8
    _ = i
}
```

- `n <= 0` → 0 lần lặp, không panic.
- `go fix` có modernizer `rangeint` tự đổi `for i := 0; i < n; i++` sang dạng này.

### 6.6 `for range` trên func / iterator (Go 1.23+)

Toán hạng là hàm có một trong ba chữ ký: `func(func() bool)`, `func(func(V) bool)` (= `iter.Seq[V]`), `func(func(K, V) bool)` (= `iter.Seq2[K, V]`).

```go
func Lines(s string) iter.Seq[string] {
    return func(yield func(string) bool) {
        for len(s) > 0 {
            line, rest, _ := strings.Cut(s, "\n")
            if !yield(line) {
                return // consumer đã dừng: dọn dẹp ở đây
            }
            s = rest
        }
    }
}

for line := range Lines(doc) {
    if line == "" {
        break // yield trả false, iterator được dịp cleanup
    }
}
```

- `break`, `return`, `goto`, `continue Outer` trong thân vòng lặp được compiler dịch thành `yield` trả về `false`; iterator **phải** dừng và `return`.
- Iterator gọi `yield` sau khi `yield` đã trả `false` → panic `runtime error: range function continued iteration after function for loop body returned false`.
- `range` trên iterator **nil** panic nil pointer dereference (khác `nil` slice/map).
- Cần kéo từng giá trị thay vì để vòng lặp đẩy? `iter.Pull(seq)` trả `next, stop`; luôn `defer stop()`.
- Stdlib có sẵn: `slices.All`/`Values`/`Sorted`, `maps.All`/`Keys`/`Values`, `strings.SplitSeq`/`FieldsSeq` (Go 1.24), `bytes.Lines`… Xem [collections.md](collections.md), [generics.md](generics.md).

### 6.7 Biến vòng lặp (Go 1.22+)

Từ Go 1.22, mọi biến khai báo bằng `:=` trong `for` (cả dạng 3 mệnh đề và `for range`) là **mới ở mỗi lần lặp** — an toàn khi capture vào goroutine/closure.

Trước 1.22 (một biến dùng chung, closure thấy giá trị cuối):

```go
// Go <= 1.21: in "3 3 3" theo thứ tự bất kỳ
for _, v := range []int{1, 2, 3} {
    go func() { fmt.Println(v) }()
}
```

Từ 1.22 (mỗi lần lặp một biến riêng):

```go
// Go >= 1.22: in 1, 2, 3 theo thứ tự bất kỳ — không cần v := v
for _, v := range []int{1, 2, 3} {
    go func() { fmt.Println(v) }()
}
```

Điều này áp dụng cả cho dạng 3 mệnh đề, kể cả khi lấy địa chỉ:

```go
var ptrs []*int
for i := 0; i < 3; i++ {
    ptrs = append(ptrs, &i)
}
// Go >= 1.22: *ptrs[0], *ptrs[1], *ptrs[2] == 0, 1, 2
// Go <= 1.21: cả ba trỏ tới cùng một biến, đều bằng 3
```

- Ngữ nghĩa mới bị **gate theo language version của từng file** (`go` directive trong `go.mod`, hoặc `//go:build go1.x` ở đầu file), không theo toolchain: module khai `go 1.21` vẫn biên dịch theo luật cũ dù build bằng go1.26. Đây là lý do một repo cũ nâng toolchain mà chưa nâng `go` directive thì không tự khỏi bug closure.
- `v := v` trong thân vòng lặp giờ là dư thừa — `go fix` có modernizer `forvar` để xóa.
- Chỉ biến khai báo **trong** `for` là per-iteration; biến khai báo trước vòng lặp vẫn dùng chung.

---

## 7. `break`, `continue` & label

```go
Outer:
for i := 0; i < 3; i++ {
    for j := 0; j < 3; j++ {
        if i == 1 && j == 1 {
            break Outer // thoát cả hai vòng
        }
        if j == 0 {
            continue // vòng trong
        }
        fmt.Println(i, j)
    }
}
```

Luật chính xác:

| Dạng | Label được gắn lên | Nếu sai |
|---|---|---|
| `break` không label | `for`/`switch`/`select` bao quanh gần nhất | — |
| `break L` | một `for`, `switch`, hoặc `select` bao quanh | `invalid break label L` |
| `continue` không label | `for` bao quanh gần nhất | không dùng được ngoài `for` |
| `continue L` | **chỉ** một `for` bao quanh | `invalid continue label L` |

- `break` trong `switch` hoặc `select` chỉ thoát `switch`/`select`, **không** thoát `for` bao ngoài — cần label nếu muốn thoát vòng. Đây là lỗi review phổ biến nhất của Go.
- Label khai báo nhưng không dùng là lỗi biên dịch: `label L defined and not used`.

```go
Loop:
for {
    switch {
    case done:
        break Loop
    default:
        step()
    }
}
```

---

## 8. `goto`

```go
func process(data []byte) (err error) {
    buf := acquire()
    if !valid(data) {
        err = ErrInvalid
        goto Release
    }
    err = decode(buf, data)
Release:
    release(buf)
    return err
}
```

Spec đặt đúng hai hạn chế, cả hai đều bị compiler bắt:

| Luật | Ví dụ sai | Thông báo compiler |
|---|---|---|
| `goto` không được làm biến mới "vào scope" | `goto L` → `v := 3` → `L:` | `goto L jumps over declaration of v` |
| `goto` ở ngoài một block không nhảy vào trong block đó | `goto L1` → `for { L1: }` | `goto L1 jumps into block starting at ...` |

- Nhảy **ra khỏi** block thì được — đó chính là mẫu "cleanup tuyến tính" ở trên.
- Label có phạm vi toàn thân hàm nhưng không xuyên biên hàm: `goto Out` trong một func literal trỏ tới label ngoài → `label Out not defined`.
- Ưu tiên cấu trúc `if`/`return`/`defer` hơn `goto`; `goto` hợp lý nhất trong parser/state machine và code sinh tự động.

---

## 9. `defer`

```go
func readAll(path string) ([]byte, error) {
    f, err := os.Open(path)
    if err != nil {
        return nil, err
    }
    defer f.Close()

    return io.ReadAll(f)
}
```

Đặc điểm quan trọng:

1. **LIFO** — defer sau chạy trước, trong phạm vi **một hàm** (không phải một block).
2. Hàm được defer và **đối số** của nó được đánh giá **ngay** lúc gặp `defer`; chỉ lời gọi bị hoãn.
3. Chạy khi hàm kết thúc — `return`, chạy hết body, hoặc panic — sau khi named result đã được gán, trước khi trả về caller.
4. Có thể đọc và sửa named result trong `defer`.

Thứ tự và thời điểm đánh giá, chạy thật trên go1.26:

```go
func demo() {
    i := 0
    defer fmt.Println("A đối số chốt sớm:", i) // đối số = 0
    defer func() { fmt.Println("B closure đọc muộn:", i) }()
    i = 42
}
// Output:
// B closure đọc muộn: 42
// A đối số chốt sớm: 0
```

`return` với named result là hai bước riêng biệt (gán result → chạy defer → trả về), nên đây là cách duy nhất "chỉnh" giá trị trả về sau `return`:

```go
func tracked() (err error) {
    defer func() {
        if err != nil {
            err = fmt.Errorf("tracked: %w", err) // caller thấy lỗi đã bọc
        }
    }()
    return doWork()
}
```

`recover()` chỉ có tác dụng khi được gọi **trực tiếp** bởi hàm được defer:

```go
defer func() { recover() }()          // dừng panic
defer func() { helper() }()           // KHÔNG dừng panic: recover() trong helper trả nil
```

Ba bẫy thường gặp:

- **`defer` trong vòng lặp:** `defer` gắn với hàm, không với block. `for _, p := range paths { f, _ := os.Open(p); defer f.Close() }` giữ mọi file descriptor tới khi hàm kết thúc. Sửa bằng cách tách thân vòng lặp thành hàm riêng.
- **`panic(nil)`:** từ Go 1.21, `recover()` trả `*runtime.PanicNilError`, không phải `nil` (`GODEBUG=panicnil=1` để quay lại hành vi cũ).
- **Chi phí:** compiler dùng *open-coded defer* (gần như miễn phí) nhưng bỏ tối ưu này khi `defer` nằm trong vòng lặp, khi một hàm có hơn 8 `defer`, khi thân `range`-over-func chứa `defer`, khi build `-race`, hoặc `-gcflags=-N`.

Chi tiết hơn: [functions.md](functions.md), [errors.md](errors.md).

---

## 10. `select` (tóm tắt)

`select` giống `switch` nhưng mỗi `case` là thao tác channel:

```go
select {
case msg := <-in:
    fmt.Println("recv", msg)
case out <- v:
    fmt.Println("sent")
case <-time.After(time.Second):
    fmt.Println("timeout")
case <-ctx.Done():
    return ctx.Err()
default:
    // non-blocking
}
```

- **Đánh giá trước:** mọi biểu thức channel và mọi giá trị cần gửi trong **tất cả** case được đánh giá đúng một lần, theo thứ tự nguồn, khi vào `select` — trước khi chọn case. `case out <- expensive():` gọi `expensive()` dù case đó không thắng. Vế trái của `v := <-ch` thì chưa được đánh giá.
- Không có case sẵn sàng + không `default` → block.
- Nhiều case sẵn sàng → spec quy định chọn bằng **uniform pseudo-random selection**, không theo thứ tự viết. Đo 1000 vòng với hai channel luôn sẵn sàng cho tỉ lệ ~50/50 (một lần chạy: 536 / 464). Đừng dựa vào thứ tự case để ưu tiên; muốn ưu tiên thì lồng `select` với `default`.
- `nil` channel trong case không bao giờ sẵn sàng — gán `ch = nil` là idiom "tắt" một nhánh, rất hữu ích cho fan-in khi các nguồn lần lượt đóng.
- Nếu **mọi** case đều `nil` và không có `default` (hoặc `select {}`), goroutine block mãi. Chương trình chỉ có goroutine đó sẽ chết: `fatal error: all goroutines are asleep - deadlock!` với trace `goroutine 1 [select]`.
- `break` trong một case chỉ ra khỏi `select`, không thoát `for` bao ngoài — dùng label.

```go
Loop:
for {
    select {
    case v, ok := <-in:
        if !ok {
            in = nil // tắt nhánh này, các nhánh khác vẫn chạy
            continue
        }
        handle(v)
    case <-ctx.Done():
        break Loop
    }
}
```

Dùng cho fan-in, timeout, cancellation với `context`. Chi tiết: [channels.md](channels.md), [concurrency.md](concurrency.md), [context.md](context.md).

---

## 11. Empty statement

Câu lệnh rỗng là `;` thuần hoặc vị trí không có statement. Hầu như luôn xuất hiện **ngầm**, không cần viết tay:

```go
for i := 0; i < 10; i++ {
    // thân rỗng — vòng lặp chỉ để chạy post statement
}

func f() {
    for i := range 3 {
        if i == 1 {
            goto Done
        }
    }
Done: // label ngay trước } vẫn hợp lệ: statement rỗng được chèn ngầm
}
```

- Không cần viết `;` sau label ở cuối block — cả `Done:` trần và `Done: ;` đều biên dịch được.
- `select {}` và `for {}` là hai cách "block vĩnh viễn": `select {}` để goroutine ngủ (nếu là goroutine duy nhất còn sống, runtime báo `fatal error: all goroutines are asleep - deadlock!` — đây là fatal error, `recover` không bắt được), còn `for {}` thì spin đốt CPU.

---

## 12. Mẹo & best practices

- **Xử lý lỗi sớm:** `if err != nil { return ... }` thay vì lồng `else` sâu.
- **Tránh shadow `err`:** tái sử dụng cùng `err` trong một luồng xử lý tuần tự khi hợp lý.
- **`range` map:** không phụ thuộc thứ tự; cần ổn định thì lấy key, `sort`, rồi duyệt.
- **Đóng channel:** chỉ bên gửi đóng; `range` bên nhận tự kết thúc.
- **Label:** đặt tên rõ (`Outer`, `ReadLoop`) — đừng lạm dụng.
- **`select` + `default`:** cẩn thận busy-loop; thường kết hợp `default` với logic khác hoặc bỏ `default` để block đúng.
- **Go 1.22+:** tận dụng per-iteration variables và `for i := range n`; bỏ các dòng `v := v`.
- **Go 1.23+:** trả `iter.Seq`/`iter.Seq2` thay vì slice trung gian khi API là "duyệt qua"; luôn `defer stop()` khi dùng `iter.Pull`.
- **Hiện đại hóa tự động:** `go fix -diff ./...` để xem, `go fix ./...` để áp dụng (Go 1.26). Xem §14.

### Phụ lục: ví dụ kết hợp

```go
func CopyN(ctx context.Context, dst io.Writer, src io.Reader, n int64) (written int64, err error) {
    buf := make([]byte, 32*1024)
    for written < n {
        select {
        case <-ctx.Done():
            return written, ctx.Err()
        default:
        }

        nr, er := src.Read(buf[:min(int64(len(buf)), n-written)])
        if nr > 0 {
            nw, ew := dst.Write(buf[:nr])
            written += int64(nw)
            if ew != nil {
                return written, ew
            }
        }
        if er != nil {
            if errors.Is(er, io.EOF) {
                break
            }
            return written, er
        }
    }
    return written, nil
}
```

---

## 13. Bẫy statement khi review code

Danh sách kiểm tra ngắn, xếp theo tần suất gặp thực tế:


| Bẫy | Dấu hiệu | Cách đúng |
| --- | -------- | --------- |
| `break` trong `switch`/`select` mà tưởng thoát `for` | `for { switch { case x: break } }` | dùng label `break Loop` |
| Shadowing bằng `:=` trong block con | biến ngoài vẫn `nil`/zero sau khối `if` | tách khai báo, hoặc dùng `=` |
| `defer` trong vòng lặp | file descriptor / lock giữ tới cuối hàm | tách thân vòng lặp thành hàm |
| `recover()` gọi gián tiếp | panic không bị bắt | `recover()` ngay trong hàm được defer |
| Sửa `a[i]` khi `range` trên array (không pointer) | `v` vẫn là giá trị cũ | `range` trên `&a` hoặc trên slice |
| `append` trong lúc `range` slice | vòng lặp không thấy phần tử mới | tự quản index, hoặc gom rồi append sau |
| Dựa vào thứ tự case của `select` | "ưu tiên" không hoạt động | lồng `select` với `default` |
| `case out <- expensive():` | `expensive()` bị gọi dù case không thắng | tính trước, đưa vào biến |
| `switch` trên `any` với case cùng dynamic type không comparable | panic runtime | type switch, hoặc `reflect.DeepEqual` |
| `for range` trên `nil` iterator function | panic nil pointer | kiểm tra `seq != nil` |
| Package khai `go 1.21` trong `go.mod` | vẫn dùng luật loop variable cũ | nâng `go` directive |

---

## 14. Tính năng theo phiên bản


| Version | Liên quan statement |
| ------- | ------------------- |
| 1.21    | built-in `min`/`max`/`clear`; `panic(nil)` → `*runtime.PanicNilError` |
| 1.22    | `for range` trên integer; biến vòng lặp per-iteration (gate theo language version của file) |
| 1.23    | `for range` trên func (`iter.Seq`, `iter.Seq2`), `iter.Pull`; GC thu hồi timer chưa `Stop` nên `time.After` trong `select` không còn là "leak" |
| 1.24    | `strings.Lines`/`SplitSeq`/`FieldsSeq`, `bytes.Lines` để `range` trực tiếp |
| 1.25    | không có thay đổi ngôn ngữ; `(*sync.WaitGroup).Go`; `testing/synctest` chính thức để test code có `select`/timer |
| 1.26    | `new(expr)`; `go fix` thành nơi chứa modernizer (`rangeint`, `forvar`, `minmax`, `waitgroup`, `stringsbuilder`…) |

Chạy `go tool fix help` để xem toàn bộ modernizer, `go fix -diff ./...` để xem patch trước khi áp dụng.

---

## Tài liệu liên quan

- [keywords.md](keywords.md) — từng từ khóa
- [operators.md](operators.md) — biểu thức, độ ưu tiên, so sánh
- [functions.md](functions.md) — `return`, `defer`, `init`
- [methods-interfaces.md](methods-interfaces.md) — type switch / assertion nâng cao
- [typesystem.md](typesystem.md) — comparable, typed nil, conversion
- [collections.md](collections.md) — slice/map và iterator của `slices`/`maps`
