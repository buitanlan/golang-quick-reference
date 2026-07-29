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

- Short `:=` tái sử dụng biến đã có **nếu ít nhất một tên mới** ở cùng block; nếu mọi tên đã tồn tại trong cùng block → lỗi hoặc thành assignment tùy ngữ cảnh.
- Tránh shadow `err`, `ctx`, `ok` một cách khó theo dõi.

```go
x := 1
x, y := 2, 3 // OK: y mới, x được gán lại
// x, y := 4, 5 trong cùng block nếu cả hai đã có → compile error với :=
x, y = 4, 5  // assignment thuần
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

- Không fall-through mặc định; muốn thì `fallthrough`.
- `case` là biểu thức, có thể gọi hàm.

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
- Không dùng `fallthrough` trong type switch.
- Case có thể liệt kê nhiều kiểu: `case int, int64:` → `v` mang kiểu `any`/`interface{}`.

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

### 6.5 `for range` trên integer (Go 1.22+)

```go
for i := range 5 {
    fmt.Println(i) // 0,1,2,3,4
}
```

### 6.6 Biến vòng lặp (Go 1.22+)

Từ Go 1.22, mỗi lần lặp tạo biến iteration **mới** — an toàn khi capture vào goroutine/closure:

```go
for i, v := range items {
    go func() {
        fmt.Println(i, v) // đúng phần tử tương ứng (1.22+)
    }()
}
```

Trên Go cũ hơn cần `i, v := i, v` trong vòng hoặc truyền tham số.

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

- `break`/`continue` + label: label gắn với `for`/`switch`/`select`.
- `break` trong `switch` chỉ thoát switch, không thoát `for` bao ngoài — cần label nếu muốn thoát vòng.

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

Hạn chế:

- Không nhảy vào scope có biến mới được khai báo.
- Không nhảy vào block khác tùy tiện — compiler kiểm soát.
- Ưu tiên cấu trúc `if`/`return`/`defer` hơn `goto`.

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

1. **LIFO** — defer sau chạy trước.
2. Đối số đánh giá **ngay** lúc gặp `defer`.
3. Chạy khi hàm **return** (kể cả panic), trước khi trả về caller.
4. Có thể sửa named result trong `defer`.

```go
func tracked() (err error) {
    defer func() {
        if err != nil {
            log.Printf("failed: %v", err)
        }
    }()
    return doWork()
}
```

Chi tiết hơn: [functions.md](functions.md).

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

- Không có case sẵn sàng + không `default` → block.
- Nhiều case sẵn sàng → chọn ngẫu nhiên (pseudo-random).
- `nil` channel trong case không bao giờ sẵn sàng.

Dùng cho fan-in, timeout, cancellation với `context`.

---

## 11. Empty statement

Câu lệnh rỗng là `;` thuần hoặc vị trí không có statement:

```go
for i := 0; i < 10; i++ {
}

L:
    // labeled empty — hiếm
    ;
```

Thường gặp gián tiếp qua quy tắc semicolon. Ít khi viết tường minh.

---

## 12. Mẹo & best practices

- **Xử lý lỗi sớm:** `if err != nil { return ... }` thay vì lồng `else` sâu.
- **Tránh shadow `err`:** tái sử dụng cùng `err` trong một luồng xử lý tuần tự khi hợp lý.
- **`range` map:** không phụ thuộc thứ tự; cần ổn định thì lấy key, `sort`, rồi duyệt.
- **Đóng channel:** chỉ bên gửi đóng; `range` bên nhận tự kết thúc.
- **Label:** đặt tên rõ (`Outer`, `ReadLoop`) — đừng lạm dụng.
- **`select` + `default`:** cẩn thận busy-loop; thường kết hợp `default` với logic khác hoặc bỏ `default` để block đúng.
- **Go 1.22+:** tận dụng per-iteration variables và `for i := range n`.

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
            if er == io.EOF {
                break
            }
            return written, er
        }
    }
    return written, nil
}
```

---

## Tài liệu liên quan

- [keywords.md](keywords.md) — từng từ khóa
- [functions.md](functions.md) — `return`, `defer`, `init`
- [methods-interfaces.md](methods-interfaces.md) — type switch / assertion nâng cao
