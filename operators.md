# Toán tử trong Go

Go có tập toán tử gọn, cố định — **không** overload operator (trừ một số ngữ nghĩa built-in như `+` cho string).  
Không có toán tử ba ngôi `?:`; dùng `if` hoặc hàm nhỏ.

---

## Mục lục

- [Toán tử trong Go](#toán-tử-trong-go)
  - [Mục lục](#mục-lục)
  - [1. Tổng quan](#1-tổng-quan)
  - [2. Bảng ưu tiên (precedence)](#2-bảng-ưu-tiên-precedence)
  - [3. Toán tử số học](#3-toán-tử-số-học)
  - [4. So sánh](#4-so-sánh)
  - [5. Logic \& short-circuit](#5-logic--short-circuit)
  - [6. Bitwise](#6-bitwise)
  - [7. Gán \& gán hợp](#7-gán--gán-hợp)
  - [8. Con trỏ: `&` và `*`](#8-con-trỏ--và-)
  - [9. Channel: `<-`](#9-channel-)
  - [10. Toán tử khác liên quan biểu thức](#10-toán-tử-khác-liên-quan-biểu-thức)
  - [11. Không có ternary — pattern thay thế](#11-không-có-ternary--pattern-thay-thế)
  - [12. Best practices](#12-best-practices)
  - [13. Toán tử trong generic (type parameter)](#13-toán-tử-trong-generic-type-parameter)
  - [14. Tính năng theo phiên bản](#14-tính-năng-theo-phiên-bản)

---

## 1. Tổng quan

- Toán tử hoạt động trên kiểu built-in theo quy tắc ngôn ngữ; không định nghĩa `operator +` cho struct tùy ý.
- `+` cũng dùng nối `string`.
- So sánh `==` yêu cầu kiểu **comparable** — xem [typesystem.md](typesystem.md).
- Logic `&&` / `||` **short-circuit**; `&` / `|` **không** dùng được trên `bool`.
- Channel ops và pointer ops là một phần cú pháp cốt lõi.
- Trên type parameter, toán tử nào dùng được phụ thuộc constraint — xem §13.

```go
a, b := 3, 4
sum := a + b
ok := a < b && b < 10
p := &a
*p = 5
ch := make(chan int, 1)
ch <- 1
v := <-ch
```

---

## 2. Bảng ưu tiên (precedence)

Từ **cao → thấp** (theo đặc tả Go):

| Mức | Toán tử | Kết hợp |
|---|---|---|
| 5 | `*` `/` `%` `<<` `>>` `&` `&^` | trái |
| 4 | `+` `-` `\|` `^` | trái |
| 3 | `==` `!=` `<` `<=` `>` `>=` | trái |
| 2 | `&&` | trái |
| 1 | `\|\|` | trái |

Chỉ có **5 mức** cho toán tử nhị phân — ít hơn C nhiều, nên dễ nhớ nhưng cũng dễ sai nếu mang thói quen C sang (trong Go `&` cùng mức với `*`, `|` cùng mức với `+`).

Unary (mức cao hơn mọi toán tử nhị phân): `+ - ! ^ * & <-`. Trên nữa là các **primary expression** (postfix, gắn chặt nhất): `x[i]`, `s[i:j]`, `x.f`, `x.(T)`, `f()`, `T(v)`, `f[T]` (instantiation generic).

Vì `*` và `&` unary yếu hơn selector, `*p.f` nghĩa là `*(p.f)` chứ không phải `(*p).f`.

Gán (`=`, `+=`, …) và `:=` không phải “biểu thức” theo nghĩa trả giá trị — không xâu chuỗi như `a = b = c` trong C.

```go
// a = b = 1  // KHÔNG hợp lệ trong Go
a := 1
b := a
```

> Khi nghi ngờ: **thêm ngoặc**. Độ ưu tiên bit vs cộng dễ gây nhầm nếu quen C.

```go
x := 1 + 2<<2   // 1 + (2<<2) = 9  — << cao hơn +
y := (1 + 2) << 2
```

---

## 3. Toán tử số học

Nhị phân: `+` `-` `*` `/` `%`  
Đơn ngôi: `+` `-`

```go
fmt.Println(7/2)    // 3 — chia nguyên, truncate về 0
fmt.Println(7/2.0)  // 3.5
fmt.Println(7%2)    // 1
fmt.Println(-7/2)   // -3 — KHÔNG phải -4: truncate về 0, không floor
fmt.Println(-7%2)   // -1 (dấu theo dividend)
fmt.Println(7%-2)   // 1  (dấu theo dividend, không theo divisor)
```

Quan hệ bất biến: `(a/b)*b + a%b == a` với `a/b` luôn **truncate về 0** (khác Python, khác toán tử `mod` toán học).

- `%` chỉ cho số nguyên (float → `invalid operation: operator % not defined on a (variable of type float64)`); với float dùng `math.Mod`.
- **Chia cho 0:** hằng 0 → lỗi biên dịch `invalid operation: division by zero`; biến 0 → `panic: runtime error: integer divide by zero`. Float chia 0 cho `+Inf`/`NaN`, không panic.
- **Tràn integer:** wrap theo số bit của kiểu (two's complement), **không** panic:

```go
var u uint8 = 0
u--                 // 255
var i8 int8 = 127
i8++                // -128
```

- **Ngoại lệ duy nhất của phép chia:** `MinInt / -1` không biểu diễn được nên spec quy định kết quả bằng chính `MinInt` (tràn two's complement), và `MinInt % -1 == 0`. Không panic, không báo lỗi:

```go
x := math.MinInt
fmt.Println(x/-1 == math.MinInt) // true
fmt.Println(x % -1)              // 0
```

- `+` trên string: nối; toán hạng phải cùng kiểu string (typed).

```go
s := "Go" + "lang"
// s2 := "Go" + 1 // LỖI
```

Nối chuỗi trong vòng lặp bằng `+=` là O(n²) vì mỗi lần cấp lại bộ đệm; dùng `strings.Builder` (hoặc `fmt.Fprintf` vào Builder). `go fix` có modernizer `stringsbuilder` tự chuyển mẫu `s += ...` trong vòng lặp sang `strings.Builder`:

```go
var b strings.Builder
b.Grow(estimate) // biết trước kích thước thì Grow một lần
for _, part := range parts {
    b.WriteString(part)
}
s := b.String()
```

Tăng/giảm: **statement** `++` / `--`, không phải biểu thức:

```go
i := 0
i++
// j := i++ // LỖI — không dùng trong biểu thức
// ++i     // LỖI — chỉ postfix statement
```

---

## 4. So sánh

`==` `!=` `<` `<=` `>` `>=`

Kết quả so sánh là **untyped bool**, nên gán được vào bất kỳ kiểu boolean nào (`type Flag bool; var f Flag = 1 < 2`).

```go
fmt.Println(3 == 3)
fmt.Println("a" < "b") // thứ tự lexicographical theo BYTE
```

Áp dụng:

- Số, string, bool (`==`/`!=`), pointer, channel, interface, array/struct comparable.
- Slice/map/func: chỉ so với `nil` bằng `==`/`!=`, **không** so hai slice với nhau bằng `==` (`invalid operation: a == b (slice can only be compared to nil)`).

```go
var s []int
fmt.Println(s == nil) // true
// fmt.Println(s == []int{}) // LỖI
fmt.Println(slices.Equal(s, []int{})) // true — dùng slices.Equal (Go 1.21+)
```

Dùng `slices.Equal` / `slices.EqualFunc` cho slice, `maps.Equal` cho map (cả hai từ Go 1.21), `bytes.Equal` cho `[]byte`; `reflect.DeepEqual` chỉ khi thật cần so sánh sâu và chấp nhận chậm.

**So sánh chuỗi là bytewise**, không theo locale và không theo thứ tự Unicode code point của ngôn ngữ tự nhiên:

```go
fmt.Println("Z" < "a")   // true  — 'Z'=0x5A < 'a'=0x61
fmt.Println("é" > "z")   // true  — "é" là 0xC3 0xA9 trong UTF-8
fmt.Println("ab" < "abc") // true — tiền tố ngắn hơn thì nhỏ hơn
```

Cần thứ tự "đúng ngôn ngữ" thì dùng `golang.org/x/text/collate`. Cần so sánh có dấu tiếng Việt theo bảng chữ cái thì **không** dùng `<` trực tiếp.

**Float và NaN** phá vỡ trực giác của `==`, kéo theo cả struct/array chứa float:

```go
fmt.Println(math.NaN() == math.NaN()) // false

type P struct{ X float64 }
a := P{math.NaN()}
fmt.Println(a == a) // false! struct chứa NaN không bằng chính nó

m := map[P]int{a: 1}
fmt.Println(len(m), m[a]) // 1 0 — chèn được nhưng không tra lại được
```

- Với float, dùng `math.IsNaN`, so sánh epsilon, hoặc `cmp.Compare` — doc của nó ghi rõ: "a NaN is considered less than any non-NaN, a NaN is considered equal to a NaN, and -0.0 is equal to 0.0".
- Đừng bao giờ dùng float (hoặc struct chứa float) làm key map nếu có khả năng gặp `NaN`.

**`==` trên interface có thể panic lúc chạy.** Compiler chỉ kiểm tra tính comparable của kiểu **tĩnh**; nếu cả hai dynamic type giống nhau và không comparable thì panic:

```go
var x, y any = []int{1}, []int{1}
_ = x == y // panic: runtime error: comparing uncomparable type []int
```

- Dynamic type **khác** nhau → trả `false`, không panic.
- Điều này áp dụng cả cho `switch x { case y: }` và cho việc dùng `any` làm key map (`panic: runtime error: hash of unhashable type []int`).

Interface và typed nil:

```go
var err error = nil
fmt.Println(err == nil) // true

var p *os.PathError = nil
var e error = p
fmt.Println(e == nil) // false — boxed typed nil!
```

> “Typed nil” trong interface là bẫy kinh điển: dynamic type ≠ nil. Xem chi tiết và cách phòng trong [typesystem.md](typesystem.md).

**Thay thế cho chuỗi so sánh dài:** `cmp.Compare` (Go 1.21) và `cmp.Or` (Go 1.22).

```go
// Sắp xếp nhiều tiêu chí: cmp.Or trả về giá trị non-zero đầu tiên
slices.SortFunc(users, func(a, b User) int {
    return cmp.Or(
        cmp.Compare(a.LastName, b.LastName),
        cmp.Compare(a.FirstName, b.FirstName),
        cmp.Compare(a.ID, b.ID),
    )
})

// cmp.Or cũng thay cho chuỗi "lấy giá trị đầu tiên khác zero"
host := cmp.Or(flagHost, envHost, "localhost")
```

`cmp.Or[T comparable](vals ...T) T` đánh giá **mọi** đối số (không short-circuit) — đừng đặt lời gọi đắt hoặc có side effect vào đó.

---

## 5. Logic & short-circuit

`&&` `||` `!` — toán hạng kiểu `bool`.

```go
if p != nil && p.Ready() {
	// p.Ready() chỉ gọi khi p != nil
}

if idle || refresh() {
	// refresh() bỏ qua nếu idle == true
}
```

- **Short-circuit**: giống C#/Java — vế phải có thể không được đánh giá. `&&` trả về sớm khi vế trái `false`, `||` khi vế trái `true`.
- Go **không** định nghĩa `&`, `|`, `^` trên `bool` (khác C/Java). Không có phiên bản "logic nhưng đánh giá đủ" của `&&`/`||`:

```go
// var c = a & b // LỖI: invalid operation: operator & not defined on a (variable of type bool)
```

Muốn buộc đánh giá cả hai vế thì tính trước vào biến:

```go
okA, okB := checkA(), checkB() // cả hai luôn chạy
if okA && okB { /* ... */ }
```

- Cẩn thận: đối số hàm **luôn** được đánh giá hết. Một helper kiểu `ifelse(cond, a(), b())` không short-circuit — xem §11.

```go
a, b := true, false
fmt.Println(a && b) // false
fmt.Println(!a)     // false
```

---

## 6. Bitwise

| Toán tử | Nghĩa |
|---|---|
| `&` | AND |
| `\|` | OR |
| `^` | XOR (nhị phân) hoặc NOT bit (đơn ngôi) |
| `&^` | bit clear (`AND NOT`) — toán tử riêng của Go, `x &^ y` = `x & (^y)` |
| `<<` `>>` | dịch trái / phải |

Cả sáu chỉ định nghĩa cho **integer** (không cho bool, không cho float, không cho string).

```go
var x uint = 0b1111_0000
fmt.Printf("%08b\n", x&0x0F)  // 00000000
fmt.Printf("%08b\n", x|0x0F)  // 11111111
fmt.Printf("%08b\n", x^0xFF)  // 00001111
fmt.Printf("%08b\n", x&^0xF0) // 00000000 — clear bits
fmt.Printf("%08b\n", ^x)      // đảo bit theo độ rộng kiểu
```

`^x` (unary NOT) đảo bit theo **độ rộng của kiểu**, nên kết quả phụ thuộc kiểu: `^int8(5)` là `-6`, còn `^uint8(5)` là `250`.

Dịch bit:

```go
fmt.Println(1<<10)      // 1024
fmt.Println(0xF0 >> 4)  // 15
```

Luật dịch bit — bốn điểm dễ sai, tất cả đều đúng theo spec và đã kiểm chứng trên go1.26:

- **Logic vs arithmetic phụ thuộc dấu của toán hạng trái**, không phụ thuộc toán tử: signed → arithmetic shift (nhân bản bit dấu), unsigned → logical shift. `int32(-8) >> 1` là `-4`; `uint32(0x80000000) >> 31` là `1` nhưng `int32(-1) >> 31` là `-1`.
- **Không có giới hạn trên cho shift count**, và **không** có hành vi undefined như C. Spec định nghĩa dịch bằng cách dịch 1 bit `n` lần, nên dịch bằng/vượt độ rộng vẫn cho kết quả xác định: với `u` kiểu `uint32` thì `u >> 40 == 0` và `u << 40 == 0`; với `i` kiểu `int32` âm thì `i >> 40 == -1` (bit dấu lấp hết).
- **Shift count được phép là signed integer** từ Go 1.13 — `1 << n` với `n int` là hợp lệ. Nếu count **âm lúc chạy** thì `panic: runtime error: negative shift amount`; nếu là hằng âm thì lỗi biên dịch `invalid operation: negative shift count -1 (untyped int constant)`.
- `go vet` (analyzer `shift`) cảnh báo khi hằng shift count vượt độ rộng kiểu, ví dụ `s (32 bits) too small for shift of 40` — hữu ích vì kết quả 0/-1 gần như luôn là bug.

```go
var x int = 1
x <<= 64 // trên nền 64-bit: x == 0, không panic
```

`go vet` bắt đúng trường hợp trên: `x (64 bits) too small for shift of 64`.

`&^` tiện cho flag:

```go
const (
	Readable = 1 << iota
	Writable
	Executable
)

perms := Readable | Writable
perms = perms &^ Writable // tắt Writable
```

---

## 7. Gán & gán hợp

```go
x := 1      // short declaration (không phải toán tử gán thuần)
x = 2
x += 3      // x = x + 3
x -= 1
x *= 2
x /= 2
x %= 2
x &= 0xFF
x |= 0x01
x ^= 0x01
x <<= 1
x >>= 1
x &^= 0x01
```

Đa gán (tuple assignment):

```go
a, b := 1, 2
a, b = b, a // swap không cần temp
v, ok := m[k]
v, err := f()
```

- Số biến bên trái = số giá trị bên phải (hoặc bên phải là một lời gọi hàm trả đúng số giá trị đó).
- Tuple assignment chạy **hai pha**: pha 1 đánh giá mọi biểu thức bên phải cùng các index/pointer indirection ở bên trái, pha 2 mới gán từ trái sang phải. Đó là lý do `a, b = b, a` hoán đổi đúng, và ví dụ của chính spec:

```go
x := []int{1, 2, 3}
i := 0
i, x[i] = 1, 2 // set i = 1, x[0] = 2 — index dùng i CŨ
i = 0
x[i], i = 2, 1 // set x[0] = 2, i = 1
```

- `:=` yêu cầu **ít nhất một biến mới trong block hiện tại**; các tên còn lại chỉ được gán lại.

```go
x, err := f()
// x, err := g() // LỖI: no new variables on left side of :=
y, err := g()    // OK — y mới, err được gán lại
x, err = g()     // OK — assignment thuần
```

Chi tiết về scope và bẫy shadowing: [statements.md](statements.md).

---

## 8. Con trỏ: `&` và `*`

```go
x := 10
p := &x   // address of x → *int
fmt.Println(*p) // 10 — dereference
*p = 20
fmt.Println(x)  // 20
```

- Go **không** có pointer arithmetic: không `p++`, không `p + 1`. Muốn cộng offset thì phải qua `unsafe.Add(ptr, len)` với đầy đủ rủi ro của `unsafe` (xem [typesystem.md](typesystem.md)).
- `&` lấy địa chỉ của **addressable value**. Bảng nhanh:

| Biểu thức | `&` được? |
|---|---|
| biến, field của biến, phần tử slice (`&s[i]`) | Có |
| phần tử array qua biến hoặc pointer (`&a[i]`, `&pa[i]`) | Có |
| composite literal (`&T{...}`, `&[]int{...}`, `&map[K]V{...}`) | Có — spec cho phép, tạo biến ẩn |
| phần tử map (`&m["a"]`) | **Không** — `cannot take address of m["a"]` |
| byte của string (`&s[0]`) | **Không** — string bất biến |
| phần tử của composite literal (`&[3]int{}[0]`) | **Không** |
| kết quả hàm (`&f()`), hằng (`&42`) | **Không** |

```go
p := &struct{ N int }{N: 1} // OK — composite literal addressable
// q := &42                 // LỖI: cannot take address of 42 (untyped int constant)
r := new(42)                // Go 1.26: cách đúng để có *int trỏ tới 42
```

- Từ Go 1.26, `new(expr)` thay thế hầu hết các trường hợp trước đây phải viết `tmp := expr; p := &tmp`. Xem [keywords.md](keywords.md).
- `&` trên composite literal của array cho `*[n]T`, và `*[n]T` index/`range`/`len`/`cap` được trực tiếp (`pa[1]`, `len(pa)`).

Method receiver pointer vs value → xem [methods-interfaces.md](methods-interfaces.md) và [pointers.md](pointers.md); toán tử vẫn là `&`/`*`.

---

## 9. Channel: `<-`

```go
ch := make(chan string, 1)

ch <- "ping"   // send
msg := <-ch    // receive
<-ch           // nhận và bỏ giá trị (nếu còn)

// close
close(ch)
v, ok := <-ch  // ok=false nếu closed và hết dữ liệu
```

Hướng trong kiểu:

```go
func produce(out chan<- int) { out <- 1 }
func consume(in <-chan int)  { <-in }
```

- `<-ch` trong `select` để chờ đa kênh.
- Gửi vào channel đã close → **panic**.
- Nhận từ nil channel / gửi vào nil channel → **block mãi**.

```go
select {
case ch <- 1:
case <-time.After(time.Second):
case <-ctx.Done():
	return
}
```

---

## 10. Toán tử khác liên quan biểu thức

Không phải “operator table” đầy đủ nhưng hay gặp trong biểu thức:

| Cú pháp | Vai trò |
|---|---|
| `x[i]` | index (slice/array/`*array`/map/string) |
| `s[i:j]`, `s[i:j:k]` | slice, slice có cap |
| `x.f` | selector field/method |
| `x.(T)` | type assertion (dạng comma-ok: `v, ok := x.(T)`) |
| `f()` | call |
| `f[int]`, `T[int]` | instantiation generic (Go 1.18+) |
| `T(v)` | conversion |
| `<-ch` | receive |
| `*` / `&` | pointer |

Precedence của indexing/call/selector/assertion cao nhất (postfix), gắn chặt hơn unary.

```go
*p.f      // *(p.f) nếu f là pointer field — đọc rõ bằng ngoặc
(*p).f
<-ch.done // <-(ch.done)
```

Lưu ý ngữ pháp: `x.(T)` và `T(v)` giống hình thức "gọi hàm" nhưng không phải operator — không có short-circuit, không có overload. Type assertion thất bại ở dạng một giá trị thì **panic**; dùng dạng comma-ok để tránh.

---

## 11. Không có ternary — pattern thay thế

Go **cố ý** không có `cond ? a : b`.

```go
// Thay bằng if
var status string
if code == 200 {
	status = "ok"
} else {
	status = "fail"
}
```

Hoặc hàm generic nhỏ:

```go
func ifelse[T any](cond bool, a, b T) T {
	if cond {
		return a
	}
	return b
}

status := ifelse(code == 200, "ok", "fail")
```

> **Cảnh báo:** helper này **không** short-circuit. `ifelse(p != nil, p.Name, "")` vẫn dereference `p` vì đối số hàm luôn được đánh giá trước khi gọi. Đây là lý do chính Go không thêm ternary — không có cách nào để một hàm mô phỏng đúng ngữ nghĩa lười của `?:`.

Ba pattern an toàn hơn, phủ hầu hết trường hợp thật:

1. Zero value + `if` — đặt mặc định trước, ghi đè sau:

```go
msg := "guest"
if name != "" {
	msg = name
}
```

2. `cmp.Or` (Go 1.22) — "giá trị non-zero đầu tiên":

```go
msg := cmp.Or(name, "guest")
```

3. `min` / `max` (Go 1.21) — thay `if`/`else` so sánh:

```go
capped := min(n, limit)
clamped := max(0, min(n, limit))
```

`min`/`max` là **built-in function**, không phải toán tử: nhận ≥ 1 đối số cùng kiểu ordered, hoạt động cả với `string`, và là biểu thức hằng nếu mọi đối số là hằng. `go fix` có modernizer `minmax` tự chuyển `if a > b { m = a } else { m = b }` sang `max(a, b)`.

Vẫn không có ternary trong Go 1.26, và cũng không có đề xuất nào được chấp nhận.

---

## 12. Best practices

- Dùng ngoặc khi trộn bit và số học — trong Go `&` cùng mức với `*`, `|` cùng mức với `+`.
- Ưu tiên `&&`/`||` để tránh nil deref nhờ short-circuit; đừng thay bằng helper hàm.
- Nhớ `==` trên interface: có thể **panic** lúc chạy, và typed nil ≠ `nil`.
- Không dùng `==` cho float sau khi tính toán, và không dùng float/struct-chứa-float làm key map.
- Kiểm tra shift count trước khi dùng biến làm số dịch; bật `go vet` (analyzer `shift` đã có trong bộ mặc định).
- Nối chuỗi trong vòng lặp → `strings.Builder`, không `+=`.
- `++`/`--` chỉ là statement — đừng tìm ternary/`i++` trong biểu thức.
- Channel: luôn rõ ai `close`; dùng `context` + `select` cho timeout/cancel.
- Không “thông minh hóa” bằng `&`/`*` rối — API nên rõ value vs pointer receiver.

Bảng tra nhanh:

| Nhóm | Toán tử |
|---|---|
| Số học | `+ - * / %` `++ --` (stmt) |
| So sánh | `== != < <= > >=` |
| Logic | `&& \|\| !` (chỉ trên `bool`) |
| Bit | `& \| ^ &^ << >>` (chỉ trên integer) |
| Gán | `= := += -= *= /= %= &= \|= ^= <<= >>= &^=` |
| Pointer | `& *` |
| Channel | `<-` |
| Không tồn tại | `?:`, `**`, pointer arithmetic, operator overload, `&` / `\|` trên `bool`, `a = b = c` |

---

## 13. Toán tử trong generic (type parameter)

Toán tử dùng được trên một type parameter `T` chỉ khi **mọi kiểu trong type set của constraint** đều hỗ trợ toán tử đó. Đây là chỗ hay bị vấp khi viết code generic:

```go
// OK: + có trên mọi kiểu của cmp.Ordered (int, float, string)
func Sum[T cmp.Ordered](xs []T) T {
	var t T
	for _, x := range xs {
		t += x
	}
	return t
}

// LỖI: % không có trên float/string
// func Mod[T cmp.Ordered](a, b T) T { return a % b }
// invalid operation: operator % not defined on a (variable of type T constrained by cmp.Ordered)
```

- `cmp.Ordered` cho `< <= > >= == !=` và `+` (vì `string` cũng có `+`), nhưng `-`, `*`, `/`, `%`, `&` đều là **lỗi biên dịch** trên `T cmp.Ordered` — thông báo có dạng `operator - not defined on a (variable of type T constrained by cmp.Ordered)`.
- Cần số học đầy đủ thì viết constraint riêng: `interface{ ~int | ~int8 | ... | ~float64 }`.
- Cần bitwise thì constraint chỉ gồm integer: `interface{ ~int | ~uint | ~uint8 | ... }`.
- `comparable` cho `==`/`!=` nhưng **không** cho `<`. Từ Go 1.20, interface cũng thỏa `comparable`, nên `==` trên `T comparable` vẫn có thể panic lúc chạy nếu `T` được instantiate bằng interface.
- Chi tiết constraint và type set: [generics.md](generics.md), [typesystem.md](typesystem.md).

---

## 14. Tính năng theo phiên bản


| Version | Liên quan toán tử / biểu thức |
| ------- | ----------------------------- |
| 1.13    | shift count được phép là signed integer; literal `0b`, `0o`, `_` trong số |
| 1.18    | generics: toán tử trên type parameter phụ thuộc type set; cú pháp instantiation `f[T]` |
| 1.20    | interface thỏa `comparable` → `==` trên `T comparable` có thể panic lúc chạy |
| 1.21    | built-in `min`/`max`/`clear`; `cmp.Ordered`, `cmp.Compare`, `cmp.Less`; `slices.Equal`, `maps.Equal` |
| 1.22    | `cmp.Or` thay chuỗi "giá trị non-zero đầu tiên" |
| 1.26    | `new(expr)` thay mẫu "biến tạm rồi `&`"; `go fix` chứa modernizer `minmax`, `stringsbuilder`, `newexpr` |

Không có toán tử mới nào được thêm vào Go từ Go 1 — mọi cải thiện đều đến từ built-in function và package (`cmp`, `slices`, `maps`).

---
