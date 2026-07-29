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

---

## 1. Tổng quan

- Toán tử hoạt động trên kiểu built-in theo quy tắc ngôn ngữ; không định nghĩa `operator +` cho struct tùy ý.
- `+` cũng dùng nối `string`.
- So sánh `==` yêu cầu kiểu **comparable** (xem typesystem).
- Logic `&&` / `||` **short-circuit**.
- Channel ops và pointer ops là một phần cú pháp cốt lõi.

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

Unary (mức cao hơn nhị phân): `+ - ! ^ * & <-` (và nhận `*` / `&` / `<-` theo ngữ cảnh).

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
fmt.Println(7/2)    // 3 — chia nguyên
fmt.Println(7/2.0)  // 3.5
fmt.Println(7%2)    // 1
fmt.Println(-7 % 2) // -1 (dấu theo dividend)
```

- `%` chỉ cho số nguyên (không dùng cho float).
- Tràn integer: wrap theo số bit của kiểu (two's complement), **không** panic.
- `+` trên string: nối; toán hạng phải cùng kiểu string (typed).

```go
s := "Go" + "lang"
// s2 := "Go" + 1 // LỖI
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

```go
fmt.Println(3 == 3)
fmt.Println("a" < "b") // thứ tự lexicographical theo byte
```

Áp dụng:

- Số, string, bool (`==`/`!=`), pointer, channel, interface, array/struct comparable.
- Slice/map/func: chỉ so với `nil` bằng `==`/`!=`, **không** so hai slice với nhau bằng `==`.

```go
var s []int
fmt.Println(s == nil) // true
// fmt.Println(s == []int{}) // LỖI
```

Interface:

```go
var err error = nil
fmt.Println(err == nil) // true

var p *os.PathError = nil
var e error = p
fmt.Println(e == nil) // false — boxed typed nil!
```

> “Typed nil” trong interface là bẫy kinh điển: dynamic type ≠ nil.

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

- **Short-circuit**: giống C#/Java — vế phải có thể không được đánh giá.
- Không có `&` / `|` logic không short-circuit trên bool ( `&` `|` trên bool vẫn có nhưng ít dùng; với bool chúng không short-circuit theo nghĩa “bitwise/full eval” — thực tế nên dùng `&&` `||`).

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
| `&^` | bit clear (`AND NOT`) |
| `<<` `>>` | dịch trái / phải |

```go
var x uint = 0b1111_0000
fmt.Printf("%08b\n", x&0x0F)  // 00000000
fmt.Printf("%08b\n", x|0x0F)  // 11111111
fmt.Printf("%08b\n", x^0xFF)  // 00001111
fmt.Printf("%08b\n", x&^0xF0) // 00000000 — clear bits
fmt.Printf("%08b\n", ^x)      // đảo bit theo độ rộng kiểu
```

Dịch bit:

```go
fmt.Println(1<<10)      // 1024
fmt.Println(0xF0 >> 4)  // 15
```

- Với unsigned: `>>` logic; signed: arithmetic shift (giữ dấu).
- Số dịch phải là unsigned integer hoặc untyped (quy tắc chi tiết trong spec); typed negative shift → lỗi biên dịch.

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

- Số biến bên trái = số giá trị bên phải.
- `:=` tái khai báo được nếu ít nhất một biến **mới** trong cùng scope.

```go
x, err := f()
x, err := g() // OK nếu err đã có — tái dùng err, nhưng cần ít nhất 1 tên mới?
// Thực ra: trong cùng block, := yêu cầu ít nhất một biến mới.
y, err := g() // OK — y mới
```

---

## 8. Con trỏ: `&` và `*`

```go
x := 10
p := &x   // address of x → *int
fmt.Println(*p) // 10 — dereference
*p = 20
fmt.Println(x)  // 20
```

- Go **không** có pointer arithmetic (trừ `unsafe`).
- `&` lấy địa chỉ của addressable value (biến, field addressable, slice element, …).
- Literal thường không addressable trực tiếp — dùng temporary hoặc composite:

```go
p := &struct{ N int }{N: 1} // OK
// q := &42                 // LỖI
```

Method receiver pointer vs value → xem OOP/methods (ngoài file này); toán tử vẫn là `&`/`*`.

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
| `x[i]` | index |
| `s[i:j]` | slice |
| `x.f` | selector field/method |
| `f()` | call |
| `T(v)` | conversion |
| `<-ch` | receive |
| `*` / `&` | pointer |

Precedence của indexing/call/selector cao (postfix), gắn chặt hơn unary.

```go
*p.f      // *(p.f) nếu f là pointer field — đọc rõ bằng ngoặc
(*p).f
```

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

Hoặc hàm nhỏ / map:

```go
func ifelse[T any](cond bool, a, b T) T {
	if cond {
		return a
	}
	return b
}

status := ifelse(code == 200, "ok", "fail")
```

Hoặc trong Go 1.22+ vẫn không có ternary ngôn ngữ — giữ `if`.

Với khởi tạo:

```go
msg := "guest"
if name != "" {
	msg = name
}
```

---

## 12. Best practices

- Dùng ngoặc khi trộn bit và số học.
- Ưu tiên `&&`/`||` để tránh nil deref nhờ short-circuit.
- Nhớ `==` trên interface và typed nil.
- `++`/`--` chỉ là statement — đừng tìm ternary/`i++` trong biểu thức.
- Channel: luôn rõ ai `close`; dùng `context` + `select` cho timeout/cancel.
- Không “thông minh hóa” bằng `&`/`*` rối — API nên rõ value vs pointer receiver.

Bảng tra nhanh:

| Nhóm | Toán tử |
|---|---|
| Số học | `+ - * / %` `++ --` (stmt) |
| So sánh | `== != < <= > >=` |
| Logic | `&& \|\| !` |
| Bit | `& \| ^ &^ << >>` |
| Gán | `= := += -= *= /= %= &= \|= ^= <<= >>= &^=` |
| Pointer | `& *` |
| Channel | `<-` |

---
