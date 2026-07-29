# Literal & Constant trong Go

**Literal** là giá trị viết trực tiếp trong mã nguồn. **Constant** (`const`) là giá trị biết lúc biên dịch — mạnh hơn nhiều ngôn ngữ nhờ **untyped constants** và `iota`.

---

## Mục lục

- [Literal \& Constant trong Go](#literal--constant-trong-go)
  - [Mục lục](#mục-lục)
  - [1. Tổng quan](#1-tổng-quan)
  - [2. Integer literals](#2-integer-literals)
  - [3. Floating-point literals](#3-floating-point-literals)
  - [4. Imaginary \& complex](#4-imaginary--complex)
  - [5. Rune literals](#5-rune-literals)
  - [6. String literals \& raw string](#6-string-literals--raw-string)
  - [7. Numeric separators `_`](#7-numeric-separators-)
  - [8. Untyped constants \& độ chính xác](#8-untyped-constants--độ-chính-xác)
  - [9. Typed vs untyped](#9-typed-vs-untyped)
  - [10. `iota`](#10-iota)
  - [11. Constant expressions](#11-constant-expressions)
  - [12. Composite literals](#12-composite-literals)
  - [13. `new(expr)` — địa chỉ giá trị (Go 1.26)](#13-newexpr--địa-chỉ-giá-trị-go-126)
  - [14. Struct tag (cú pháp)](#14-struct-tag-cú-pháp)
  - [15. Mẹo \& lỗi thường gặp](#15-mẹo--lỗi-thường-gặp)

---

## 1. Tổng quan

```go
const MaxRetry = 3
var port = 8080          // biến, không phải const
msg := "hello"           // string literal → biến typed string
```

- Literal xuất hiện trong biểu thức; kiểu có thể là **untyped** cho tới khi “neo” vào ngữ cảnh.
- `const` chỉ cho giá trị tính được lúc compile (số, rune, string, bool, và biểu thức từ chúng — kể cả một số `len` của array/string constant…).
- Slice/map/channel/`new(...)` / gọi hàm thường **không** phải constant (dù kết quả `new` có thể trỏ tới giá trị khởi tạo từ literal).

Xem hệ thống kiểu & default type: [typesystem.md §2.4](typesystem.md#24-untyped-constant--default-type).

---

## 2. Integer literals

Cơ số:

| Dạng | Ví dụ | Ghi chú |
|---|---|---|
| Thập phân | `42`, `1000` | mặc định |
| Nhị phân | `0b1010`, `0B1010` | Go 1.13+ |
| Octal | `0o755`, `0O755` | Go 1.13+; legacy `0755` vẫn hợp lệ nhưng dễ nhầm |
| Hex | `0xFF`, `0Xff` | |

```go
const (
	dec = 42
	bin = 0b_0010_1010
	oct = 0o755
	hex = 0x2A
	legacyOct = 0755 // = 493 thập phân — cân nhắc tránh
)
```

- Không có suffix `U`/`L` như C#; kích thước/kiểu phụ thuộc **ngữ cảnh gán** hoặc khai báo typed const.
- Integer constant trong spec có **độ chính xác tùy ý**; implementation phải biểu diễn được ít nhất **256 bit** cho số nguyên (xem §8).

---

## 3. Floating-point literals

```go
const (
	pi       = 3.1416
	avogadro = 6.022e23
	small    = 1.2e-3
	dotStart = .5
	dotEnd   = 5.
	hexFloat = 0x1.fp3   // = 15.5 (Go 1.13+ hexadecimal float)
	half     = 0x1p-2    // = 0.25 — mantissa hex, exponent lũy thừa 2
)
```

- Dạng thập phân: phần nguyên/phân số với `.` và/hoặc mũ `e`/`E`.
- Hex float (1.13+): tiền tố `0x`/`0X`, mantissa hex (có thể có `.`), bắt buộc mũ `p`/`P` (lũy thừa **2**, không phải 10). Ví dụ: `0x1p-2 == 0.25`, `0x1.fp3 == 15.5`.
- Untyped float constant → default type `float64` khi cần typed float mặc định.

---

## 4. Imaginary & complex

```go
const (
	i  = 1i
	c  = 1 + 2i
	c2 = complex(3, 4) // const nếu cả hai args là constant
)
```

- Literal ảo kết thúc bằng `i` (có thể kết hợp với float/hex float: `0x1p-2i`).
- Kind untyped complex; khi neo: default type `complex128` (không phải `complex64`).

---

## 5. Rune literals

Rune literal trong dấu nháy đơn `'...'` — một Unicode code point:

```go
const (
	a   = 'A'          // rune / untyped rune
	nl  = '\n'
	tab = '\t'
	sq  = '\''
	bs  = '\\'
	hex = '\x41'       // byte hex → 'A'
	u16 = '\u65e5'     // 日
	u32 = '\U0001F600' // U+1F600
)
```

Escape phổ biến: `\a \b \f \n \r \t \v \\ \' \"`.

Thêm:

| Dạng | Ví dụ | Ý nghĩa |
|---|---|---|
| Octal byte | `'\377'` | tối đa 3 chữ số octal |
| Hex byte | `'\xFF'` | đúng 2 chữ số hex |
| Unicode | `'\uXXXX'` | 4 hex → code point |
| Unicode dài | `'\UXXXXXXXX'` | 8 hex → code point |

- Kiểu mặc định khi cần typed: `rune` (= `int32`).
- Khác string: `'A'` là số (code point), `"A"` là chuỗi UTF-8 (1 byte với ASCII).

```go
var r rune = '日'
var s string = "日"
fmt.Println(r, []rune(s))
```

---

## 6. String literals & raw string

### Interpreted string `"..."`

```go
s := "hello\nworld"
path := "C:\\Users\\go" // phải escape \
quote := "say \"hi\""
hex := "\x41"           // "A"
u := "\u65e5"           // "日"
```

- Hỗ trợ escape giống phần lớn C-family (kể cả `\x`, `\u`, `\U`).
- Nội dung phải là chuỗi byte UTF-8 hợp lệ (compiler kiểm tra).
- Byte riêng lẻ: `"A"[0]` có kiểu `byte`, không phải `rune`.

### Raw string `` `...` ``

```go
path := `C:\Users\go`
sql := `
SELECT id, name
FROM users
WHERE active = 1
`
regex := `^[a-z]+\d*$`
```

- Không xử lý escape (trừ việc kết thúc bởi backtick).
- Có thể nhiều dòng; newline được giữ nguyên.
- Không chứa được backtick `` ` `` bên trong — phải nối hoặc dùng interpreted string.

```go
s := "raw with ` tick"
```

---

## 7. Numeric separators `_`

Go 1.13+: dấu `_` giữa các chữ số để dễ đọc:

```go
const (
	million = 1_000_000
	mask    = 0b_1111_0000
	addr    = 0xFF_EC_DE_5E
	afterPx = 0x_10         // OK: _ ngay sau prefix, rồi chữ số
	pi      = 3.141_592
	avog    = 6.022_140_76e+23
	hexF    = 0x1.0p-1_023  // separator trong phần mũ cũng được nếu giữa chữ số
)
```

Quy tắc (spec):

- `_` phải **tách các chữ số liên tiếp** — không đầu/cuối literal, không `__`, không chỉ sau prefix rồi hết (`0b_` lỗi).
- Sau `0x`/`0b`/`0o` vẫn được `_` nếu tiếp theo là chữ số hợp lệ (`0x_10` OK).
- Không làm thay đổi giá trị — chỉ cosmetic.

---

## 8. Untyped constants & độ chính xác

Đây là điểm khác biệt quan trọng so với C#:

```go
const X = 1        // untyped integer constant
const S = "hi"     // untyped string constant
const B = true     // untyped boolean
const R = 'A'      // untyped rune
```

Untyped constant có **kind** (integer/float/string/…) nhưng chưa gắn named type. Có thể gán vào nhiều kiểu tương thích:

```go
const N = 100

var (
	a int     = N
	b int64   = N
	c float64 = N
	d byte    = N
)
```

Giới hạn: giá trị phải **biểu diễn được** trong kiểu đích:

```go
const Big = 1 << 100
var f float64 = Big // OK nếu vừa float64
// var i int = Big  // lỗi nếu tràn int
```

Độ chính xác (chi tiết & default type: [typesystem.md §2.4](typesystem.md#24-untyped-constant--default-type)):

- Phép toán trên untyped constant thực hiện ở **compile time** với độ chính xác tùy ý — `2.0/3.0` không phải chia `float64`.
- Spec yêu cầu implementation biểu diễn integer constant ≥ **256 bit**; float/complex constant ít nhất bằng precision của `float64`/`complex128` khi cần.
- Tràn kiểu đích → **lỗi biên dịch**, không wrap như biến runtime.

```go
const Ratio = 2.0 / 3.0 // chính xác cao
x := Ratio              // x là float64 (default type)
var f32 float32 = Ratio // làm tròn tại điểm gán
```

---

## 9. Typed vs untyped

```go
const Untyped = 1
const Typed int = 1
const Typed2 = int(1) // typed int constant
```

```go
var x int32 = Untyped // OK
// var y int32 = Typed // LỖI: Typed là int, không gán sang int32 không conversion
var y int32 = int32(Typed)
```

Khi nào dùng typed const?

- Cần API rõ kiểu (`const DefaultPort int16 = 443` — ít gặp).
- Thường **ưu tiên untyped** để linh hoạt; dùng typed khi muốn chặn nhầm lẫn.

```go
type Kind int

const (
	KindA Kind = 1
	KindB Kind = 2
)
```

Ở đây `KindA` là **typed** constant kiểu `Kind` (nhờ chuyển kiểu trong khởi tạo iota/group).

---

## 10. `iota`

`iota` sinh dãy số nguyên tăng trong **const block**, reset về 0 mỗi khối `const ( ... )`:

```go
const (
	Sunday = iota // 0
	Monday        // 1
	Tuesday       // 2
)
```

Biểu thức lặp lại ngầm:

```go
const (
	_  = iota             // bỏ 0
	KB = 1 << (10 * iota) // iota=1 → 1<<10
	MB                    // iota=2 → 1<<20
	GB                    // 1<<30
	TB                    // 1<<40
)
```

```go
type Flag uint

const (
	FlagReadable Flag = 1 << iota // 1
	FlagWritable                  // 2
	FlagExecutable                // 4
)
```

Nhiều iota trong một dòng:

```go
const (
	a, b = iota, iota // 0, 0
	c, d              // 1, 1
)
```

Bỏ giữa / nhảy giá trị:

```go
const (
	A = iota // 0
	B        // 1
	_        // 2 bị bỏ
	C        // 3
)

const (
	X = iota + 1 // 1
	Y            // 2
	Z            // 3
)
```

Custom stringer thủ công (hoặc `stringer`):

```go
type Day int

const (
	Sun Day = iota
	Mon
	Tue
)

func (d Day) String() string {
	switch d {
	case Sun:
		return "Sun"
	case Mon:
		return "Mon"
	case Tue:
		return "Tue"
	default:
		return fmt.Sprintf("Day(%d)", int(d))
	}
}
```

- `iota` chỉ trong const declaration.
- Không phải enum runtime của C#; chỉ là số/const — zero value thường mang nghĩa “chưa set” nếu bắt đầu từ 0.

---

## 11. Constant expressions

Phép toán trên const vẫn là const nếu mọi toán hạng là const:

```go
const (
	W        = 16
	H        = 9
	Pixels   = W * H
	Mask     = (1 << 8) - 1
	Greeting = "hi" + " " + "go"
	OK       = true && !false
)
```

Được phép (rút gọn): `+ - * / %`, bit ops, so sánh (cho bool const), `len`/`cap` của array hoặc string **constant**, conversion giữa kiểu số/string/rune trong quy tắc const, `real`/`imag`/`complex`, `min`/`max` (1.21+) với args const…

Không phải const:

```go
var n = 3
// const X = n + 1 // LỖI

// const Y = len([]int{1,2}) // LỖI — slice literal không cho len const
const Z = len("abc") // OK → 3
```

```go
type Vec [3]int
const Dim = len(Vec{}) // OK — array length là phần của kiểu
```

---

## 12. Composite literals

Khởi tạo giá trị kiểu **composite** (struct, array, slice, map) bằng `{...}`:

```go
type User struct {
	ID   int
	Name string
}

u := User{ID: 1, Name: "Ada"}     // keyed
u2 := User{1, "Ada"}              // unkeyed — theo thứ tự field
a := [3]int{1, 2, 3}
s := []string{"a", "b"}
m := map[string]int{"x": 1}
```

Đặc điểm quan trọng:

- Field/index **zero** có thể bỏ; keyed cho phép không theo thứ tự.
- Unkeyed struct literal **phải** liệt kê đủ field exported trong package khác — thường ưu tiên keyed.
- Lồng nhau được **elide** kiểu phần tử:

```go
type Point struct{ X, Y int }
points := []Point{{1, 2}, {3, 4}} // elide Point
grid := [][]int{{1, 2}, {3}}
```

- Lấy địa chỉ composite literal được phép — tạo biến ẩn:

```go
p := &User{ID: 1, Name: "Ada"} // *User
q := &[3]int{1, 2, 3}          // *[3]int
```

- `&` trên map/slice element hoặc giá trị trả về hàm thì **không** addressable — xem [pointers.md](pointers.md), [operators.md](operators.md).
- Từ 1.26, nhiều chỗ trước đây cần biến tạm/`&` có thể viết `new(expr)` (§13).

Parsing note: composite literal ngay sau `if`/`for`/`switch` có thể cần ngoặc `(T{})` để tránh nhầm với block.

---

## 13. `new(expr)` — địa chỉ giá trị (Go 1.26)

Trước 1.26, `new` chỉ nhận **kiểu** (`new(int)` → `*int` zero). Go 1.26 cho phép `new(expr)`: cấp phát biến kiểu của `expr`, khởi tạo bằng giá trị `expr`, trả về pointer.

```go
p := new(42)           // *int, *p == 42
s := new("hi")         // *string
b := new(true)         // *bool
u := new(User{ID: 1})  // *User — bản copy của literal

x := User{ID: 7}
q := new(x)            // *User copy của x tại thời điểm gọi
x.ID = 9               // q.ID vẫn 7
```

So với `&` trên composite:

| Cách | Ghi chú |
|---|---|
| `&User{...}` | addressable composite literal (có từ trước) |
| `new(User{...})` | tương đương ý nghĩa pointer tới bản copy |
| `tmp := 42; p := &tmp` | mẫu cũ cho non-composite |
| `p := new(42)` | 1.26 — thay mẫu biến tạm |

- Untyped constant → convert sang **default type** trước (`new(42)` → `*int`, không phải untyped).
- `new(nil)` **không** hợp lệ (nil không có kiểu).
- `new(T)` (kiểu) vẫn zero value như trước.
- Modernizer `newexpr` của `go fix` (1.26) gợi ý viết lại mẫu cũ — xem [keywords.md](keywords.md), [main-function.md](main-function.md).

Dùng điển hình: field optional dạng pointer trong JSON/protobuf:

```go
type Person struct {
	Name string `json:"name"`
	Age  *int   `json:"age"`
}

p := Person{Name: "Ada", Age: new(42)}
```

---

## 14. Struct tag (cú pháp)

Tag là **string literal** (thường raw) ngay sau field trong struct type — không phải constant độc lập, nhưng là literal quan trọng:

```go
type User struct {
	ID       int    `json:"id" db:"user_id"`
	Name     string `json:"name,omitempty"`
	Password string `json:"-"`
}
```

- Cú pháp: `` Field Type `key:"value" key2:"value2"` ``.
- Package như `encoding/json`, `reflect.StructTag` đọc tag lúc runtime.
- Chi tiết convention `json`/`xml`/`db`, `omitempty`/`omitzero`: [structs-composition.md §8](structs-composition.md#8-tag-json-xml-db).

---

## 15. Mẹo & lỗi thường gặp

- Octal legacy `0755`: dễ nhầm với thập phân — ưu tiên `0o755`.
- `"A"[0]` là `byte`; `'A'` là rune/untyped rune.
- Nối string const OK; nối với biến → không còn const.
- Dùng iota cho bitflag / enum số; nhớ zero value thường mang nghĩa “chưa set” — có thể bắt đầu từ `iota + 1`.
- Untyped float gán vào `int` chỉ khi giá trị là số nguyên chính xác:

```go
const F = 1.0
var i int = F // OK
// const G = 1.2
// var j int = G // LỖI
```

- Raw string rất tiện cho Windows path, regex, SQL, JSON mẫu.
- Hằng exported (`MaxIdle`) vs unexported (`maxIdle`) theo chữ cái đầu — giống mọi identifier.
- Cần pointer tới số/`bool`/string đơn giản → `new(expr)` (1.26) thay vì biến tạm.
- Tag struct: xem [structs-composition.md](structs-composition.md); hệ thống kiểu untyped: [typesystem.md](typesystem.md).

Bảng tóm tắt literal:

| Literal | Ví dụ | Kind mặc định (untyped) |
|---|---|---|
| int | `42`, `0xFF`, `0b1010`, `0o755` | integer |
| float | `3.14`, `1e3`, `0x1p-2` | floating-point |
| imag | `2i` | imaginary |
| rune | `'日'` | rune |
| string | `"a"`, `` `a` `` | string |
| bool | `true` | boolean |
| composite | `T{...}`, `[]int{1}`, `map[K]V{}` | typed theo `T` |
| `new(expr)` | `new(42)` | `*DefaultType` (1.26) |

### Version map

| Phiên bản | Liên quan literal / const |
|---|---|
| 1.13 | `0b`/`0o`, hex float `0x1p-2`, separator `_` |
| 1.21 | `min`/`max`/`clear` dùng được trong const expression (với args const) |
| 1.26 | `new(expr)` — pointer tới giá trị khởi tạo từ expression |

---
