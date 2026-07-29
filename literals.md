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
  - [8. Untyped constants](#8-untyped-constants)
  - [9. Typed vs untyped](#9-typed-vs-untyped)
  - [10. `iota`](#10-iota)
  - [11. Constant expressions](#11-constant-expressions)
  - [12. Mẹo \& lỗi thường gặp](#12-mẹo--lỗi-thường-gặp)

---

## 1. Tổng quan

```go
const MaxRetry = 3
var port = 8080          // biến, không phải const
msg := "hello"           // string literal → biến typed string
```

- Literal xuất hiện trong biểu thức; kiểu có thể là **untyped** cho tới khi “neo” vào ngữ cảnh.
- `const` chỉ cho giá trị tính được lúc compile (số, rune, string, bool, và biểu thức từ chúng — kể cả một số `len` của array/string constant…).
- Slice/map/channel/`new` **không** phải constant.

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
- Số nguyên untyped có độ chính xác lớn (ít nhất 256-bit trong đặc tả cho integer constant arithmetic).

---

## 3. Floating-point literals

```go
const (
	pi     = 3.1416
	avogadro = 6.022e23
	small  = 1.2e-3
	dotStart = .5
	dotEnd   = 5.
	hexFloat = 0x1.fp3   // = 15.5 (Go 1.13+ hexadecimal float)
)
```

- Dạng: thập phân với `.` và/hoặc mũ `e`/`E`.
- Hex float: `0x` mantissa + `p`/`P` exponent (lũy thừa 2).
- Untyped float constant → thường trở thành `float64` khi cần typed float mặc định.

---

## 4. Imaginary & complex

```go
const (
	i  = 1i
	c  = 1 + 2i
	c2 = complex(3, 4) // không phải const nếu args không const; với const args → const
)
```

- Literal ảo kết thúc bằng `i`.
- Kiểu untyped complex; khi neo: `complex128` phổ biến hơn `complex64`.

---

## 5. Rune literals

Rune literal trong dấu nháy đơn `'...'` — một Unicode code point:

```go
const (
	a   = 'A'      // rune / untyped rune
	nl  = '\n'
	tab = '\t'
	sq  = '\''
	bs  = '\\'
	hex = '\x41'   // byte hex → 'A'
	u16 = '\u65e5' // 日
	u32 = '\U0001F600' // 😀
)
```

Escape phổ biến: `\a \b \f \n \r \t \v \\ \' \"`.

- Kiểu mặc định khi cần typed: `rune` (= `int32`).
- Khác string: `'A'` là số (code point), `"A"` là chuỗi 1 byte (trong ASCII).

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
```

- Hỗ trợ escape giống phần lớn C-family.
- Nội dung là chuỗi byte UTF-8 hợp lệ (compiler kiểm tra).

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
	pi      = 3.141_592
	avog    = 6.022_140_76e+23
)
```

Quy tắc:

- Chỉ giữa chữ số (hoặc phần hex hợp lệ); không đầu/cuối.
- Không liền kề `__`.
- Không đặt ngay sau prefix `0x`/`0b`/`0o` rồi `_` rồi hết — phải có chữ số hai bên theo rule (ví dụ `0x_10` hợp lệ trong Go).

---

## 8. Untyped constants

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
	_ = iota // bỏ 0
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

- `iota` chỉ trong const declaration.
- Bỏ giá trị: `_ = iota` hoặc comment tường minh.
- Không phải enum runtime của C#; chỉ là số/const — thường kết hợp `String()` generated (`stringer`).

---

## 11. Constant expressions

Phép toán trên const vẫn là const nếu mọi toán hạng là const:

```go
const (
	W = 16
	H = 9
	Pixels = W * H
	Mask = (1 << 8) - 1
	Greeting = "hi" + " " + "go"
	OK = true && !false
)
```

Được phép (rút gọn): `+ - * / %` , bit ops, so sánh (cho bool const), `len` của string/array constant, conversion giữa kiểu số/string/rune trong quy tắc const, `real`/`imag`/`complex`, …

Không phải const:

```go
var n = 3
// const X = n + 1 // LỖI

// const Y = len([]int{1,2}) // LỖI — slice literal không phải const length context như vậy
const Z = len("abc") // OK → 3
```

```go
type Vec [3]int
const Dim = len(Vec{}) // OK — array length là phần của kiểu
```

---

## 12. Mẹo & lỗi thường gặp

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

Bảng tóm tắt literal:

| Literal | Ví dụ | Kind mặc định (untyped) |
|---|---|---|
| int | `42`, `0xFF` | integer |
| float | `3.14`, `1e3` | floating-point |
| imag | `2i` | imaginary |
| rune | `'日'` | rune |
| string | `"a"`, `` `a` `` | string |
| bool | `true` | boolean |

---
