# Hệ thống kiểu dữ liệu trong Go

Go là ngôn ngữ **statically typed**: mọi biến/biểu thức có kiểu xác định lúc biên dịch.  
Hệ thống kiểu đơn giản hơn C#/Java (không class hierarchy cổ điển), nhưng có **named types**, **interfaces**, **type assertion/switch**, và quy tắc chuyển đổi **tường minh**.

> Tài liệu nhắm **Go 1.26**; tính năng theo phiên bản được ghi rõ (1.18 → 1.26) — xem bảng tổng hợp ở cuối.

---

## Mục lục

- [Hệ thống kiểu dữ liệu trong Go](#hệ-thống-kiểu-dữ-liệu-trong-go)
  - [Mục lục](#mục-lục)
  - [1. Static typing — tổng quan](#1-static-typing--tổng-quan)
  - [2. Kiểu cơ bản (basic types)](#2-kiểu-cơ-bản-basic-types)
    - [2.1 bool](#21-bool)
    - [2.2 Numeric](#22-numeric)
    - [2.3 string, byte, rune](#23-string-byte-rune)
    - [2.4 Untyped constant & default type](#24-untyped-constant--default-type)
  - [3. Named type vs underlying type](#3-named-type-vs-underlying-type)
    - [3.1 Type identity & assignability](#31-type-identity--assignability)
  - [4. Type definition vs type alias](#4-type-definition-vs-type-alias)
    - [4.1 Generic type alias (Go 1.24+)](#41-generic-type-alias-go-124)
  - [5. Zero values](#5-zero-values)
  - [6. Conversion (không implicit)](#6-conversion-không-implicit)
    - [6.1 Các conversion “ít gặp” nên biết](#61-các-conversion-ít-gặp-nên-biết)
  - [7. Type assertions](#7-type-assertions)
  - [8. Type switches](#8-type-switches)
  - [9. Comparable types](#9-comparable-types)
    - [9.1 `comparable` vs “strictly comparable”](#91-comparable-vs-strictly-comparable)
    - [9.2 Bẫy NaN](#92-bẫy-nan)
  - [10. Defined types & method sets (góc nhìn)](#10-defined-types--method-sets-góc-nhìn)
  - [11. Interface ở mức runtime](#11-interface-ở-mức-runtime)
  - [12. Type parameter dưới góc nhìn type system](#12-type-parameter-dưới-góc-nhìn-type-system)
  - [13. Kiểu hàm & iterator (Go 1.23+)](#13-kiểu-hàm--iterator-go-123)
  - [14. `reflect`: kiểu ở runtime](#14-reflect-kiểu-ở-runtime)
  - [15. Bộ nhớ: stack/heap & escape analysis](#15-bộ-nhớ-stackheap--escape-analysis)
  - [16. `unsafe` (overview ngắn)](#16-unsafe-overview-ngắn)
  - [17. Bảng tra: assignable / convertible / comparable](#17-bảng-tra-assignable--convertible--comparable)
  - [18. Sơ đồ tư duy nhanh](#18-sơ-đồ-tư-duy-nhanh)

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

**Language version quyết định luật kiểu nào được bật.** Directive `go` trong `go.mod` (hoặc `//go:build go1.x` cho từng file) chọn phiên bản *ngôn ngữ*, độc lập với toolchain đang chạy:

```go
// với go.mod ghi "go 1.22", toolchain 1.26 vẫn từ chối tính năng mới hơn:
// generic type alias requires go1.23 or later (-lang was set to go1.22; check go.mod)
// cannot range over Count(3) (value of func type iter.Seq[int]): requires go1.23 or later
```

- Nâng version ngôn ngữ = nâng `go` directive, không chỉ cài Go mới.
- Xem thêm [build-constraints.md](build-constraints.md) và [packages-modules.md](packages-modules.md).

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


| Kiểu          | Mô tả                                          |
| ------------- | ---------------------------------------------- |
| `int`, `uint` | Ít nhất 32-bit; thường 64-bit trên amd64/arm64 |
| `uintptr`     | Số nguyên đủ chứa pointer (dùng với `unsafe`)  |


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


| Kiểu     | Bản chất                                   |
| -------- | ------------------------------------------ |
| `string` | Immutable sequence of bytes (thường UTF-8) |
| `byte`   | Alias `uint8`                              |
| `rune`   | Alias `int32` — Unicode code point         |


- Index `s[i]` cho **byte**, không phải ký tự Unicode.
- Độ dài `len(s)` = số byte, không phải số rune.
- Đếm rune: `utf8.RuneCountInString(s)`.

### 2.4 Untyped constant & default type

Constant không kiểu được tính với **độ chính xác tùy ý** ở compile time, chỉ nhận kiểu khi cần đặt vào một biến/biểu thức có kiểu:


| Loại constant | Default type     |
| ------------- | ---------------- |
| integer       | `int`            |
| rune (`'a'`)  | `rune` (`int32`) |
| float         | `float64`        |
| complex       | `complex128`     |
| string        | `string`         |
| boolean       | `bool`           |


```go
const Big = 1 << 62        // OK: constant, chưa có kiểu
const Huge = 1 << 200      // OK khi chỉ dùng trong biểu thức constant
var ok = Huge >> 190       // OK: kết quả biểu diễn được trong int
// var bad int = Huge      // LỖI compile: overflows int

const Ratio = 2.0 / 3.0    // tính bằng độ chính xác cao, không phải float64
x := Ratio                 // x là float64 (default type)
var f32 float32 = Ratio    // làm tròn về float32 tại đây
```

- Constant tràn kiểu → **lỗi biên dịch**, khác biến runtime (wrap around).
- `iota` cũng là untyped constant; gắn kiểu bằng `type Weekday int` + `const ( Sun Weekday = iota; ... )`.
- Chi tiết cú pháp: [literals.md](literals.md).

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

### 3.1 Type identity & assignability

Ba khái niệm dễ lẫn — phân biệt được là hiểu 90% lỗi kiểu của Go:


| Khái niệm       | Nghĩa                               | Ví dụ                                          |
| --------------- | ----------------------------------- | ---------------------------------------------- |
| **Identical**   | cùng một kiểu (alias vẫn identical) | `type A = B` → `A` ≡ `B`                       |
| **Assignable**  | gán được không cần conversion       | `[]byte` → `[]byte`, untyped const → typed var |
| **Convertible** | cần `T(v)` tường minh               | `int` → `int64`, `[]byte` → `string`           |


Quy tắc assignability hay dùng (rút gọn):

1. Cùng kiểu → OK.
2. Underlying **identical** và **ít nhất một bên không phải named type** → OK.
3. Value implement interface đích → OK (boxing).
4. Untyped constant biểu diễn được trong kiểu đích → OK.
5. `nil` gán cho pointer, slice, map, chan, func, interface → OK.
6. Channel: gán bidirectional `chan T` cho `<-chan T` / `chan<- T` → OK.

```go
type Ints []int

var a Ints
var b []int

a = b        // OK: []int là unnamed type, underlying identical (luật 2)
b = a        // OK: cùng lý do

type Nums []int
// a = Nums{} // LỖI: hai named type khác nhau → cần Ints(Nums{})
```

```go
ch := make(chan int)
var send chan<- int = ch // OK
var recv <-chan int = ch // OK
// ch = send             // LỖI: không đi ngược được
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


|              | Definition `type A B`     | Alias `type A = B`                                                    |
| ------------ | ------------------------- | --------------------------------------------------------------------- |
| Kiểu mới?    | Có                        | Không — cùng kiểu                                                     |
| Method riêng | Có thể gắn method cho `A` | Method của `B`; không tạo method set “mới” cho alias như defined type |
| Gán với `B`  | Cần conversion            | Gán trực tiếp                                                         |


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

Lưu ý alias:

- Alias **không** cho phép gắn method lên kiểu của package khác: `type MyTime = time.Time` rồi `func (t MyTime) F()` → lỗi (`cannot define new methods on non-local type`).
- `%T` in ra kiểu gốc, không phải tên alias — alias không tồn tại ở runtime.

### 4.1 Generic type alias (Go 1.24+)

Từ Go 1.24, alias có thể nhận **type parameter** (compiler yêu cầu `-lang` ≥ `go1.23`):

```go
type Set[T comparable] = map[T]struct{}
type List[T any]       = []T
type Result[T any]     = struct {
	Value T
	Err   error
}

s := Set[string]{"a": {}}   // = map[string]struct{}
var l List[int] = []int{1}  // gán trực tiếp, không cần conversion
```

- Vì là alias, `Set[string]` **identical** với `map[string]struct{}` → interop tự nhiên với code cũ.
- Muốn đóng gói hành vi + method thì vẫn cần defined type: `type Set[T comparable] map[T]struct{}`.
- Dùng khi rút gọn kiểu generic dài dòng hoặc di chuyển kiểu giữa các package mà không phá code gọi.

---

## 5. Zero values

Mọi biến được khai báo không khởi tạo đều có **zero value** — không còn “uninitialized garbage” kiểu C:


| Kiểu                                       | Zero                 |
| ------------------------------------------ | -------------------- |
| `bool`                                     | `false`              |
| số                                         | `0` / `0.0` / `0+0i` |
| `string`                                   | `""`                 |
| pointer, slice, map, chan, func, interface | `nil`                |
| struct                                     | zero từng field      |
| array                                      | zero từng phần tử    |


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

`new(T)` trả `*T` đã zero. Từ **Go 1.26**, `new` nhận cả **biểu thức** — rất tiện cho optional field dạng con trỏ:

```go
p := new(int)          // *int trỏ tới 0
q := new(42)           // Go 1.26+: *int trỏ tới 42
s := new("hi")         // *string

type Person struct {
	Name string `json:"name"`
	Age  *int   `json:"age"` // nil = chưa biết
}
_ = Person{Name: "An", Age: new(30)} // trước 1.26 phải khai báo biến tạm rồi &tmp
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

Conversion số **không** báo lỗi khi mất dữ liệu — nó cắt bit / làm tròn im lặng:

```go
var big int32 = 300
var b byte = byte(big)        // 44 — mất bit cao, không panic
var neg int = -1
var u uint8 = uint8(neg)      // 255
var f float64 = 3.9
var i int = int(f)            // 3 — truncate về 0, không round
```

> Muốn phát hiện tràn, so sánh trước khi convert (`if v > math.MaxUint8 { ... }`) hoặc dùng `strconv.ParseInt` với bitSize.

Đừng dùng conversion để đổi số ↔ chuỗi số:

```go
n := 65
_ = string(rune(n))     // "A" — code point, hiếm khi là điều bạn muốn
// _ = string(n)        // vet: conversion from int to string yields a string of one rune
_ = strconv.Itoa(n)     // "65" — đúng ý định
```

### 6.1 Các conversion “ít gặp” nên biết

```go
b := []byte("hello")

arr := [5]byte(b)        // Go 1.20+: slice → array (panic nếu len(b) < 5)
pa := (*[2]byte)(b)      // Go 1.17+: slice → pointer-to-array (chia sẻ backing array)
_ = arr
_ = pa
// _ = [3]byte("abc")    // KHÔNG có: string → array không hợp lệ
```

Struct convertible khi các field **identical** — **tag bị bỏ qua** khi so sánh (Go 1.8+):

```go
type APIUser struct {
	Name string `json:"name"`
}
type DBUser struct {
	Name string `db:"name"`
}

var a APIUser
var d = DBUser(a) // OK: chỉ khác tag
_ = d
```

- Rất tiện để tách DTO / model mà không copy field thủ công.
- Vẫn phải cùng **thứ tự**, **tên**, **kiểu** field; thêm/bớt field là lỗi.

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
- Toán hạng **phải** là interface: `var n int; n.(int)` → lỗi biên dịch (`invalid operation: n is not an interface`).
- `x.(T)` trên interface `nil` luôn fail (kể cả `T` là interface) — comma-ok trả `false`, không panic.

```go
var err error // nil
_, ok := err.(interface{ Timeout() bool })
fmt.Println(ok) // false — nil interface không có dynamic type
```

Với error, dùng `errors.As` thay vì assertion trực tiếp để đi qua chuỗi wrap:

```go
var pathErr *fs.PathError
if errors.As(err, &pathErr) { // unwrap %w, assertion chỉ nhìn lớp ngoài
	fmt.Println(pathErr.Path)
}

// Go 1.26+: bản generic, không cần con trỏ tới biến đích
if pathErr, ok := errors.AsType[*fs.PathError](err); ok {
	fmt.Println(pathErr.Path)
}
```

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
- Case được xét **theo thứ tự**: `case error` đặt trước `case *MyErr` sẽ “ăn” hết — để interface case xuống dưới.
- `case nil` chỉ khớp interface nil (chưa có dynamic type), **không** khớp typed nil như `(*T)(nil)`.
- Trùng kiểu trong hai case → lỗi biên dịch (`duplicate case`).
- Không `switch` trực tiếp trên type parameter — phải qua `any`:

```go
func kind[T any](v T) string {
	switch any(v).(type) { // v.(type) → "cannot use type switch on type parameter value"
	case int:
		return "int"
	case string:
		return "string"
	}
	return "other"
}
```

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

Constraint `comparable` trong generics (từ Go 1.18):

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

```go
var a any = []int{1}
_ = a == a // panic: comparing uncomparable type []int
```

### 9.1 `comparable` vs “strictly comparable”

- **Strictly comparable**: `==` không bao giờ panic (số, string, pointer, chan, struct/array của các kiểu này).
- **Spec-comparable**: `==` hợp lệ về mặt kiểu nhưng có thể panic ở runtime — chính là interface.

Từ **Go 1.20**, kiểu interface *thỏa mãn* (satisfies) constraint `comparable` dù không strictly comparable:

```go
// Go 1.18/1.19: lỗi. Go 1.20+: OK
type Cache[K comparable] map[K]int
var c = Cache[any]{}
c[42] = 1
c["k"] = 2
// c[[]int{1}] = 3 // compile OK, panic ở runtime khi hash key non-comparable
```

Đổi lại, trách nhiệm chuyển sang runtime: instantiate `comparable` bằng `any` là hợp lệ nhưng bạn tự chịu rủi ro panic.

### 9.2 Bẫy NaN

`NaN != NaN` nên float làm map key rất dễ sai:

```go
nan := math.NaN()
m := map[float64]string{}
m[nan] = "a"
m[nan] = "b"
fmt.Println(len(m), nan == nan) // 2 false — mỗi lần ghi tạo một entry mới, không đọc lại được

type Point struct{ X, Y float64 }
p := Point{nan, 1}
fmt.Println(p == p) // false — struct chứa NaN không bằng chính nó
```

- Struct/array chứa float ⇒ `==` không còn là “so sánh nội dung” đáng tin.
- Cần key số thực? Chuẩn hóa trước (làm tròn về integer scale, hoặc dùng string/decimal).
- Nội dung sâu (slice/map/pointer): `reflect.DeepEqual` (chậm) hoặc so sánh thủ công.

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

## 11. Interface ở mức runtime

Interface value = **2 word**: con trỏ type descriptor (+ bảng method) và con trỏ data.

```go
var i any
var e error
var s []int
var str string
fmt.Println(unsafe.Sizeof(i), unsafe.Sizeof(e))     // 16 16 (amd64)
fmt.Println(unsafe.Sizeof(s), unsafe.Sizeof(str))   // 24 16
```


| Kiểu                     | Bố cục            | Size (64-bit) |
| ------------------------ | ----------------- | ------------- |
| `any` / interface        | `(type, data)`    | 16            |
| `string`                 | `(ptr, len)`      | 16            |
| slice                    | `(ptr, len, cap)` | 24            |
| map, chan, func, pointer | 1 con trỏ         | 8             |


**Typed nil ≠ nil interface** — bug kinh điển vì word type khác `nil`:

```go
type MyErr struct{}

func (e *MyErr) Error() string { return "boom" }

func broken() error {
	var p *MyErr = nil
	return p // interface = (*MyErr, nil) → KHÁC nil
}

func ok() error {
	return nil // interface = (nil, nil)
}

fmt.Println(broken() == nil) // false ← nguồn gốc vô số bug
fmt.Println(ok() == nil)     // true
```

Quy tắc: **đừng** khai báo biến kiểu con trỏ cụ thể rồi return nó như `error`; return `nil` tường minh.

Chi phí cần biết:

- Đóng giá trị vào interface có thể **cấp phát heap** (xem §15); giá trị nhỏ như `0`, `1`… dùng cache tĩnh của runtime.
- Gọi method qua interface là **indirect call** — chặn inline, dù compiler có devirtualize được vài trường hợp khi biết kiểu tĩnh.
- `any` giữ `(type descriptor, data)`; interface có method giữ `(itab, data)` — **itab** là cặp (interface × kiểu cụ thể) kèm bảng method, được runtime cache lại.
- Hot path: dùng generics hoặc kiểu cụ thể thay `any` nếu đo được overhead — xem [methods-interfaces.md](methods-interfaces.md).

---

## 12. Type parameter dưới góc nhìn type system

Type parameter là kiểu **compile-time**; nó không phải interface và không tồn tại ở runtime.

```go
func zero[T any]() T {
	var z T   // zero value của T
	return z
}
```

Giới hạn cần nhớ:


| Việc                                             | Được?                                                      |
| ------------------------------------------------ | ---------------------------------------------------------- |
| `var z T` (zero value)                           | Có                                                         |
| `T(v)` conversion trong thân generic             | Có nếu hợp lệ với **mọi** kiểu trong type set              |
| `v.(type)` trên giá trị kiểu `T`                 | Không — chuyển qua `any(v)` trước                          |
| Method có type parameter riêng                   | Không                                                      |
| Type parameter trong method set / interface động | Không                                                      |
| `reflect` trên `T`                               | Có, nhưng chỉ ở runtime: `reflect.TypeFor[T]()` (Go 1.22+) |


```go
func nameOf[T any]() string {
	return reflect.TypeFor[T]().String() // không cần giá trị mẫu như TypeOf
}
```

- Constraint là interface **chỉ dùng làm constraint** nếu chứa type set (`~int | ~string`) — không dùng làm kiểu biến.
- Instantiation không sinh code cho mỗi kiểu: compiler dùng **GC shape stenciling** + dictionary, nên generics không nhanh bằng code viết tay cho từng kiểu.
- Chi tiết: [generics.md](generics.md).

**Go 1.26+: constraint tự tham chiếu.** Kiểu generic được phép xuất hiện trong type parameter list của chính nó (F-bounded polymorphism) — diễn tả “T phải trả về chính kiểu T”:

```go
type Adder[A Adder[A]] interface {
	Add(A) A
}

func algo[A Adder[A]](x, y A) A { return x.Add(y) }

type MyInt int

func (m MyInt) Add(o MyInt) MyInt { return m + o }

fmt.Println(algo(MyInt(2), MyInt(3))) // 5
```

Hữu ích cho builder / fluent API và các interface “clone/merge chính kiểu mình” mà trước 1.26 phải viết bằng `any` + assertion.

---

## 13. Kiểu hàm & iterator (Go 1.23+)

Kiểu hàm là first-class; từ Go 1.23 một số **kiểu hàm nhất định** còn range được:

```go
import "iter"

// iter.Seq[V]  = func(yield func(V) bool)
// iter.Seq2[K,V] = func(yield func(K, V) bool)

func Count(n int) iter.Seq[int] {
	return func(yield func(int) bool) {
		for i := range n { // Go 1.22+: range trên int
			if !yield(i) {
				return // consumer break → dừng producer
			}
		}
	}
}

for i := range Count(3) { // 0 1 2
	fmt.Print(i, " ")
}
```

- `range` chấp nhận `func(func() bool)`, `func(func(V) bool)`, `func(func(K, V) bool)` — đúng **shape** là được, không cần tên `iter.Seq`.
- `yield` trả `false` nghĩa là consumer đã `break`/`return`; producer phải dừng.
- `maps.Keys` / `slices.Values` trả `iter.Seq` — xem [collections.md](collections.md).
- `iter.Pull` biến push-iterator thành pull (`next, stop := iter.Pull(seq)`).

---

## 14. `reflect`: kiểu ở runtime

Khi type switch không đủ (decode động, ORM, serializer):

```go
t := reflect.TypeOf(v)         // nil nếu v là nil interface
t = reflect.TypeFor[int]()     // Go 1.22+: không cần giá trị

fmt.Println(t.Name(), t.Kind()) // "int" int
```

Phân biệt **Type** và **Kind**:

```go
type UserID int64

t := reflect.TypeOf(UserID(1))
fmt.Println(t.Name())   // UserID  ← named type
fmt.Println(t.Kind())   // int64   ← underlying kind
```

- `Kind` là tập hữu hạn (`Int64`, `Slice`, `Struct`, `Ptr`, …) ↔ đúng khái niệm underlying type ở §3.
- Ghi giá trị cần `reflect.Value` **addressable + exported**: `reflect.ValueOf(&x).Elem().SetInt(7)`.
- Đọc tag: `t.Field(i).Tag.Get("json")`.
- Go 1.26+: duyệt bằng iterator thay vòng `for i := 0; i < t.NumField(); i++`:

```go
for f := range reflect.TypeFor[Server]().Fields() { // iter.Seq[reflect.StructField]
	fmt.Println(f.Name, f.Type, f.Tag.Get("json"))
}
// tương tự: Type.Methods(), Type.Ins(), Type.Outs(), Value.Fields(), Value.Methods()
```

- Reflect chậm và mất type safety lúc compile — ưu tiên generics/interface, chỉ reflect ở biên hệ thống (encoding, config, DI).

---

## 15. Bộ nhớ: stack/heap & escape analysis

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

func boxed() any {
	x := 42
	return x // boxing vào interface cũng escape
}
```

Xem phân tích:

```bash
go build -gcflags="-m" .     # thêm -m -m để biết lý do chi tiết
```

Output thật của ví dụ trên:

```text
./main.go:6:2:  moved to heap: x
./main.go:17:9: 42 escapes to heap
./main.go:21:13: ... argument does not escape
```

Các nguyên nhân escape hay gặp:


| Tình huống                                               | Vì sao                                                                          |
| -------------------------------------------------------- | ------------------------------------------------------------------------------- |
| Return `&x`                                              | pointer sống lâu hơn frame                                                      |
| Gán vào interface / `any`                                | data word cần địa chỉ ổn định                                                   |
| `fmt.Println(x)`                                         | `...any` boxing (thường compiler giữ mảng argument trên stack, giá trị vẫn box) |
| Capture trong closure gửi vào goroutine                  | closure sống ngoài frame                                                        |
| Slice/map lớn hoặc kích thước không xác định lúc compile | không đặt được trên stack                                                       |
| Lưu vào field của object đã ở heap                       | reachability                                                                    |


High-level:

- **Stack**: frame hàm, tự thu hồi khi return; stack goroutine tự lớn dần (copy stack).
- **Heap**: GC (concurrent mark-sweep, non-moving cho object người dùng) quản lý.
- Go 1.22+: biến vòng lặp **per-iteration** — capture trong closure an toàn hơn, đổi lại có thể tạo nhiều allocation hơn nếu biến escape mỗi vòng.
- Không có destructor. `runtime.SetFinalizer` khó dùng đúng (giữ object sống thêm 1 chu kỳ, không chạy nếu có vòng tham chiếu); **Go 1.24+ ưu tiên `runtime.AddCleanup`**.

```go
type File struct{ fd int }

func open(fd int) *File {
	f := &File{fd: fd}
	// cleanup nhận arg riêng, KHÔNG giữ tham chiếu tới f
	runtime.AddCleanup(f, func(fd int) { syscall.Close(fd) }, f.fd)
	return f
}
```

Cleanup/finalizer chỉ là lưới an toàn — API tốt vẫn cần `Close()` tường minh + `defer`.

Hai công cụ liên quan tới kiểu & bộ nhớ (Go 1.23/1.24):

```go
// unique (1.23): interning giá trị comparable → so sánh & so khớp bằng 1 con trỏ
type Key struct{ Ns, Name string }
h1 := unique.Make(Key{"a", "b"})
h2 := unique.Make(Key{"a", "b"})
fmt.Println(h1 == h2)   // true, so sánh O(1) thay vì so 2 string
fmt.Println(h1.Value()) // {a b}

// weak (1.24): tham chiếu yếu, không chặn GC — dùng cho cache
wp := weak.Make(obj)
if p := wp.Value(); p != nil {
	use(p)
}
```

---

## 16. `unsafe` (overview ngắn)

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

`Sizeof` / `Alignof` / `Offsetof` là hằng compile-time — hữu ích để xem **padding**:

```go
type S struct {
	A bool  // offset 0
	B int64 // offset 8 (padding 7 byte sau A)
	C bool  // offset 16
}
// unsafe.Sizeof(S{}) == 24; sắp lại A, C cạnh nhau → 16
```

- Sắp field từ **lớn → nhỏ** để giảm padding khi struct nhiều và nóng.
- `unsafe.Sizeof` là kích thước **shallow**: `Sizeof([]byte{...})` luôn 24, không tính backing array.

Go 1.20+ có API an toàn hơn cho việc “nhìn” bytes và string, thay cho hack `reflect.StringHeader`:

```go
b := []byte("hello")

s := unsafe.String(unsafe.SliceData(b), len(b)) // []byte → string, KHÔNG copy
bs := unsafe.Slice(unsafe.StringData(s), len(s)) // string → []byte, KHÔNG copy
_ = bs
```

- Hợp đồng: sau khi tạo `string` bằng `unsafe.String`, **không được ghi** vào `b` nữa (string phải bất biến).
- Đây là con dao hai lưỡi cho hot path parse/serialize; benchmark trước, và đừng để zero-copy string thoát ra API public.
- Interop C/hệ điều hành: đặt marker `_ structs.HostLayout` (Go 1.23+) ở đầu struct để yêu cầu layout theo host ABI:

```go
type winRect struct {
	_                      structs.HostLayout
	Left, Top, Right, Bottom int32
}
```

- Kiểm tra code `unsafe` bằng `go vet` (`unsafeptr`) và chạy test với `-race`, `GOEXPERIMENT=checkptr` (bật mặc định khi `-race`).

---

## 17. Bảng tra: assignable / convertible / comparable


| Cặp kiểu                             | Gán trực tiếp      | `T(v)`               | `==`                    |
| ------------------------------------ | ------------------ | -------------------- | ----------------------- |
| `int` ↔ `int64`                      | Không              | Có                   | Có (sau convert)        |
| `int` ↔ `float64`                    | Không              | Có (truncate)        | Có (sau convert)        |
| `UserID` (def. `int64`) ↔ `int64`    | Không              | Có                   | Có (sau convert)        |
| `MyInt` (alias `int`) ↔ `int`        | Có                 | Có (no-op)           | Có                      |
| `[]byte` ↔ `string`                  | Không              | Có (copy)            | Không (`[]byte`)        |
| `Ints` (def. `[]int`) ↔ `[]int`      | Có (1 bên unnamed) | Có                   | Không                   |
| `[]byte` → `[5]byte`                 | Không              | Có (1.20+)           | Array: Có               |
| struct chỉ khác tag                  | Không              | Có (1.8+)            | Có nếu field comparable |
| `chan T` → `<-chan T`                | Có                 | Có                   | Có (pointer identity)   |
| Kiểu cụ thể → interface nó implement | Có                 | Có                   | Có (có thể panic)       |
| interface → kiểu cụ thể              | Không              | Không (dùng `x.(T)`) | —                       |
| slice / map / func                   | —                  | —                    | Chỉ so với `nil`        |


---

## 18. Sơ đồ tư duy nhanh

```
Types
├── Basic: bool, numeric, string
├── Composite: array, struct, slice, map, channel, function, pointer, interface
├── Defined types (type T U) + methods
├── Aliases (type T = U, có type params từ 1.24)
└── Type params (generics)

Values
├── Zero value luôn xác định
├── Untyped constant → default type khi cần
├── Conversion tường minh T(v)
└── Interface: (type, data) → assertion / type switch
```

Checklist thực tế:

- Cần phân biệt ngữ nghĩa ID? → defined type.
- Cần key map? → comparable **và** không chứa float/NaN.
- Nhận `any`? → assertion comma-ok hoặc type switch; với `error` dùng `errors.As`.
- Return `error`? → đừng return con trỏ nil (typed nil, §11).
- Convert số? → kiểm tra tràn trước; số ↔ chuỗi dùng `strconv`.
- Rút gọn kiểu generic dài? → generic type alias (1.24+).
- Nghi escape/GC? → `-gcflags='-m -m'`, giảm boxing vào `any` ở hot path.
- Cần dọn resource? → `Close()` + `defer`; cùng lắm mới `runtime.AddCleanup`.

### Tính năng theo phiên bản


| Version | Liên quan hệ thống kiểu                                                                              |
| ------- | ---------------------------------------------------------------------------------------------------- |
| 1.9     | type alias `type A = B`                                                                              |
| 1.17    | slice → pointer-to-array                                                                             |
| 1.18    | generics, `any`, `comparable`                                                                        |
| 1.20    | interface thỏa `comparable`; slice → array; `unsafe.String/Slice/StringData/SliceData`               |
| 1.21    | `min`/`max`/`clear`, `cmp.Ordered`, suy luận kiểu tốt hơn                                            |
| 1.22    | `for range` trên int, biến vòng lặp per-iteration, `reflect.TypeFor`                                 |
| 1.23    | range over func (`iter.Seq`), `unique.Handle`, `structs.HostLayout`                                  |
| 1.24    | generic type alias, `runtime.AddCleanup`, `weak.Pointer`                                             |
| 1.25    | spec bỏ khái niệm “core type”, diễn đạt lại luật cho slice/map/chan generic                          |
| 1.26    | `new(expr)`, constraint tự tham chiếu `type A[T A[T]]`, `errors.AsType[E]`, iterator trong `reflect` |


---

