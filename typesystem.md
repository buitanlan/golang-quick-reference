# Hệ thống kiểu dữ liệu trong Go

Go là ngôn ngữ **statically typed**: mọi biến/biểu thức có kiểu xác định lúc biên dịch.  
Hệ thống kiểu đơn giản hơn C#/Java (không class hierarchy cổ điển), nhưng có **named types**, **interfaces**, **type assertion/switch**, và quy tắc chuyển đổi **tường minh**.

---

## Mục lục

- [Hệ thống kiểu dữ liệu trong Go](#hệ-thống-kiểu-dữ-liệu-trong-go)
  - [Mục lục](#mục-lục)
  - [1. Static typing — tổng quan](#1-static-typing--tổng-quan)
  - [2. Kiểu cơ bản (basic types)](#2-kiểu-cơ-bản-basic-types)
    - [2.1 bool](#21-bool)
    - [2.2 Numeric](#22-numeric)
    - [2.3 string, byte, rune](#23-string-byte-rune)
  - [3. Named type vs underlying type](#3-named-type-vs-underlying-type)
  - [4. Type definition vs type alias](#4-type-definition-vs-type-alias)
  - [5. Zero values](#5-zero-values)
  - [6. Conversion (không implicit)](#6-conversion-không-implicit)
  - [7. Type assertions](#7-type-assertions)
  - [8. Type switches](#8-type-switches)
  - [9. Comparable types](#9-comparable-types)
  - [10. Defined types \& method sets (góc nhìn)](#10-defined-types--method-sets-góc-nhìn)
  - [11. Bộ nhớ: stack/heap \& escape analysis](#11-bộ-nhớ-stackheap--escape-analysis)
  - [12. `unsafe` (overview ngắn)](#12-unsafe-overview-ngắn)
  - [13. Sơ đồ tư duy nhanh](#13-sơ-đồ-tư-duy-nhanh)

---

## 1. Static typing — tổng quan

- Kiểu được kiểm tra lúc **compile**; sai kiểu → không ra binary.
- Có **type inference** khi khai báo với `:=` hoặc `var x = expr`, nhưng kiểu vẫn cố định sau đó.
- Không có boxing tự động kiểu C#; `interface{}` / `any` là “hộp” tường minh.
- Generics (Go 1.18+): `func Min[T cmp.Ordered](a, b T) T` — tham số kiểu được kiểm tra tĩnh.

```go
var n int = 10
m := 20          // m có kiểu int (untyped constant 20 → int theo ngữ cảnh)
// n = "x"       // lỗi biên dịch
```

---

## 2. Kiểu cơ bản (basic types)

### 2.1 bool

```go
var ok bool      // false
flag := true
```

- Chỉ `true` / `false`. **Không** ép từ số (khác C).

### 2.2 Numeric

**Integer (architecture-dependent size):**

| Kiểu | Mô tả |
|---|---|
| `int`, `uint` | Ít nhất 32-bit; thường 64-bit trên amd64/arm64 |
| `uintptr` | Số nguyên đủ chứa pointer (dùng với `unsafe`) |

**Integer kích thước cố định:**

`int8` `int16` `int32` `int64`  
`uint8` `uint16` `uint32` `uint64`

**Floating & complex:**

`float32` `float64`  
`complex64` `complex128`

```go
var (
	i   int     = 42
	u   uint    = 42
	f   float64 = 3.14
	c   complex128 = 1 + 2i
	b   byte    = 255 // alias của uint8
	r   rune    = '日'  // alias của int32 (code point Unicode)
)
```

- Không có `decimal` trong stdlib core; dùng `math/big` hoặc thư viện tiền tệ khi cần chính xác thập phân.
- Tràn số nguyên: **wrap around** (không panic mặc định), khác `checked` của C#.

### 2.3 string, byte, rune

```go
s := "Xin chào"     // string = chuỗi byte UTF-8 bất biến
bs := []byte(s)     // copy sang slice byte
rs := []rune(s)     // code points

for i, r := range s { // range theo rune, i là byte index
	_ = i
	_ = r
}
```

| Kiểu | Bản chất |
|---|---|
| `string` | Immutable sequence of bytes (thường UTF-8) |
| `byte` | Alias `uint8` |
| `rune` | Alias `int32` — Unicode code point |

- Index `s[i]` cho **byte**, không phải ký tự Unicode.
- Độ dài `len(s)` = số byte, không phải số rune.

---

## 3. Named type vs underlying type

Mỗi kiểu có **underlying type** (kiểu nền):

```go
type UserID int64        // named/defined type; underlying = int64
type Name string

var id UserID = 10
var n int64 = 10
// id = n              // LỖI: UserID và int64 khác named type
id = UserID(n)         // OK: conversion tường minh
```

- Hai biến cùng underlying **không** gán trực tiếp nếu named type khác nhau.
- Untyped constants có thể linh hoạt hơn khi gán vào typed variable (nếu biểu diễn được).

```go
const Answer = 42      // untyped integer constant
var x UserID = Answer  // OK
var y int32 = Answer   // OK
```

---

## 4. Type definition vs type alias

```go
// Type definition (tạo kiểu mới / defined type)
type Celsius float64
type Fahrenheit float64

// Type alias (tên khác của cùng một kiểu) — Go 1.9+
type MyInt = int
type any = interface{} // từ Go 1.18, any là alias built-in
```

Khác biệt:

| | Definition `type A B` | Alias `type A = B` |
|---|---|---|
| Kiểu mới? | Có | Không — cùng kiểu |
| Method riêng | Có thể gắn method cho `A` | Method của `B`; không tạo method set “mới” cho alias như defined type |
| Gán với `B` | Cần conversion | Gán trực tiếp |

```go
type Celsius float64

func (c Celsius) String() string {
	return fmt.Sprintf("%.1f°C", c)
}

type C = Celsius // alias
var t C = 36.5
_ = t.String()
```

Dùng alias khi: refactor đổi tên dần, hoặc export cùng kiểu với tên khác.  
Dùng definition khi: muốn phân biệt ngữ nghĩa (`UserID` ≠ `int64`) hoặc gắn method.

---

## 5. Zero values

Mọi biến được khai báo không khởi tạo đều có **zero value** — không còn “uninitialized garbage” kiểu C:

| Kiểu | Zero |
|---|---|
| `bool` | `false` |
| số | `0` / `0.0` / `0+0i` |
| `string` | `""` |
| pointer, slice, map, chan, func, interface | `nil` |
| struct | zero từng field |
| array | zero từng phần tử |

```go
var p *int          // nil
var s []int         // nil slice (len=0, cap=0)
var m map[string]int // nil map — đọc OK (miss), ghi sẽ panic
var f func()        // nil — gọi sẽ panic
```

```go
type Server struct {
	Addr string
	Port int
}
var srv Server // Addr="", Port=0
```

---

## 6. Conversion (không implicit)

Go **không** chuyển đổi số / string ngầm giữa typed values:

```go
var i int = 10
var f float64 = float64(i) // bắt buộc
var u uint = uint(i)

var s string = string(rune(65)) // "A" — string(int) theo code point nếu là integer type
var b []byte = []byte("hi")
var t string = string(b)
```

- Conversion `T(v)` hợp lệ khi quy tắc ngôn ngữ cho phép (cùng underlying, số ↔ số, slice ↔ string trong một số case, …).
- **Không** nhầm conversion với type assertion (`x.(T)` chỉ cho interface).

```go
type ID int
var id ID = ID(42)
var n int = int(id)
```

---

## 7. Type assertions

Dùng khi giá trị có kiểu interface và bạn muốn lấy dynamic type:

```go
var i any = "hello"

s := i.(string)        // panic nếu không phải string
s, ok := i.(string)    // comma-ok: ok=false nếu sai, không panic
if ok {
	fmt.Println(s)
}

type Stringer interface{ String() string }
var v any = time.Second
if st, ok := v.(fmt.Stringer); ok {
	fmt.Println(st.String())
}
```

- Dạng một kết quả: panic nếu fail.
- Dạng `value, ok`: an toàn — luôn dùng trong code production khi không chắc.
- Assertion sang interface khác: kiểm tra method set.

---

## 8. Type switches

```go
func describe(v any) string {
	switch x := v.(type) {
	case nil:
		return "nil"
	case int:
		return fmt.Sprintf("int=%d", x)
	case string:
		return fmt.Sprintf("string=%q", x)
	case []byte:
		return fmt.Sprintf("bytes len=%d", len(x))
	case error:
		return "error: " + x.Error()
	default:
		return fmt.Sprintf("%T", x)
	}
}
```

- `v.(type)` **chỉ** trong `switch`.
- Mỗi `case` có thể liệt kê nhiều kiểu: `case int, int64:`.
- Trong `case T`, biến `x` có kiểu `T` (hoặc interface nếu nhiều type trong một case).

---

## 9. Comparable types

Kiểu **comparable** mới dùng được với `==` / `!=` và làm key của `map`:

Comparable (rút gọn):

- boolean, numeric, string, pointer, channel
- interface (so sánh dynamic type + value; cẩn thận)
- struct nếu **mọi** field comparable
- array nếu phần tử comparable

**Không** comparable:

- slice, map, function
- struct chứa field không comparable

```go
m := map[string]int{"a": 1} // OK
// map[[]byte]int{}         // LỖI

type Point struct{ X, Y int }
_ = map[Point]bool{} // OK

type Bad struct{ Data []byte }
// _ = map[Bad]bool{} // LỖI
```

Go 1.20+: constraint `comparable` trong generics:

```go
func Keys[M ~map[K]V, K comparable, V any](m M) []K {
	keys := make([]K, 0, len(m))
	for k := range m {
		keys = append(keys, k)
	}
	return keys
}
```

> So sánh interface: nếu dynamic value không comparable → **panic** lúc runtime.

---

## 10. Defined types & method sets (góc nhìn)

```go
type Counter int

func (c *Counter) Inc() { *c++ }

func (c Counter) Value() int { return int(c) }
```

- Method gắn với **defined type** (hoặc pointer tới nó), không gắn trực tiếp lên predeclared `int`.
- Method set quyết định implement interface:

```go
type Valuer interface{ Value() int }

var c Counter = 3
var v Valuer = c  // OK — Value có receiver giá trị
// var x interface{ Inc() } = c // LỖI — Inc cần *Counter
var x interface{ Inc() } = &c   // OK
```

---

## 11. Bộ nhớ: stack/heap & escape analysis

Go **không** cho lập trình viên chọn stack/heap trực tiếp bằng từ khóa như `new` vs stack alloc tường minh trong mọi trường hợp.

- `new(T)`, `&T{}`, `make` — cấp phát; compiler quyết định region qua **escape analysis**.
- Nếu địa chỉ biến **thoát** khỏi hàm (return pointer, đóng vào goroutine, lưu vào heap object…) → thường lên **heap**.
- Nếu không escape → có thể ở **stack** → rẻ hơn, không áp lực GC.

```go
func answer() *int {
	x := 42
	return &x // x escape → heap
}

func local() int {
	x := 42
	return x // không escape
}
```

Xem phân tích:

```bash
go build -gcflags="-m" .
```

High-level:

- **Stack**: frame hàm, tự thu hồi khi return.
- **Heap**: GC quản lý; pointer giữa object được theo dõi.
- Không có destructors; có `runtime.SetFinalizer` (hiếm khi nên dùng).

---

## 12. `unsafe` (overview ngắn)

Package `unsafe` cho phép phá vỡ type safety có chủ đích:

```go
import (
	"fmt"
	"unsafe"
)

func main() {
	var x int64 = 1
	fmt.Println(unsafe.Sizeof(x))  // 8
	fmt.Println(unsafe.Alignof(x))
	_ = unsafe.Pointer(&x)
}
```

- `unsafe.Pointer` ↔ `uintptr` cho arithmetic địa chỉ (rất dễ sai / bị GC “làm hỏng” nếu giữ `uintptr` lâu).
- Dùng trong runtime, serialization tối ưu, interop — **không** phải công cụ hàng ngày.
- Code `unsafe` có thể phụ thuộc phiên bản compiler/GC; đọc kỹ release notes.

---

## 13. Sơ đồ tư duy nhanh

```
Types
├── Basic: bool, numeric, string
├── Composite: array, struct, slice, map, channel, function, pointer, interface
├── Defined types (type T U) + methods
├── Aliases (type T = U)
└── Type params (generics)

Values
├── Zero value luôn xác định
├── Conversion tường minh T(v)
└── Interface: assertion / type switch
```

Checklist thực tế:

- Cần phân biệt ngữ nghĩa ID? → defined type.
- Cần key map? → đảm bảo comparable.
- Nhận `any`? → assertion comma-ok hoặc type switch.
- Nghi escape/GC? → `-gcflags=-m`, giảm allocation hot path.

---
