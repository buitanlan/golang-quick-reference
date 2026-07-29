# Method & Interface trong Go

---

## Mục lục

1. [Method — khái niệm](#1-method--khái-niệm)
2. [Receiver: value vs pointer](#2-receiver-value-vs-pointer)
3. [Method set (`T` vs `*T`)](#3-method-set-t-vs-t)
4. [Embedding & method promotion](#4-embedding--method-promotion)
5. [Interface — thỏa mãn ngầm](#5-interface--thỏa-mãn-ngầm)
6. [`any` / empty interface](#6-any--empty-interface)
7. [Type assertion](#7-type-assertion)
8. [Interface value: (type, value)](#8-interface-value-type-value)
9. [Nil interface vs nil pointer](#9-nil-interface-vs-nil-pointer)
10. [Tránh interface pollution](#10-tránh-interface-pollution)
11. [Mẫu `io.Reader`](#11-mẫu-ioreader)
12. [`comparable` & constraint (Go 1.18 / 1.20+)](#12-comparable--constraint-go-118--120)
13. [Best practices](#13-best-practices)

---

## 1. Method — khái niệm

Method là hàm gắn với **receiver** thuộc một kiểu:

```go
type Counter struct {
    n int
}

func (c Counter) Value() int { // value receiver
    return c.n
}

func (c *Counter) Inc() { // pointer receiver
    c.n++
}
```

Gọi:

```go
var c Counter
c.Inc()          // Go tự lấy &c nếu cần
fmt.Println(c.Value())
```

- Method chỉ khai báo trong **cùng package** với kiểu receiver (không thêm method cho `int`, `time.Time` từ package khác).
- Có thể đặt method cho mọi kiểu định nghĩa bởi `type` trong package (struct, slice named, func named…).

```go
type Degrees float64

func (d Degrees) Radians() float64 {
    return float64(d) * math.Pi / 180
}
```

Không có inheritance class kiểu C#/Java — tái sử dụng qua **embedding** và **interface**.

---

## 2. Receiver: value vs pointer

### 2.1 Value receiver

```go
func (p Point) Dist() float64 {
    return math.Hypot(p.X, p.Y)
}
```

- Làm việc trên **bản sao**.
- Phù hợp kiểu nhỏ, bất biến logic, không cần mutate.

### 2.2 Pointer receiver

```go
func (p *Point) Move(dx, dy float64) {
    p.X += dx
    p.Y += dy
}
```

- Có thể sửa field; tránh copy lớn.
- **Nil receiver** hợp lệ nếu method xử lý được:

```go
func (s *Stack) Len() int {
    if s == nil {
        return 0
    }
    return len(s.items)
}
```

### 2.3 Quy ước nhất quán

Nếu **một** method của kiểu dùng pointer receiver, thường để **mọi** method mutating/API chính dùng pointer — tránh lẫn lộn method set (mục sau).

---

## 3. Method set (`T` vs `*T`)

**Method set** quyết định kiểu có thỏa interface nào và gọi method nào được.

| Kiểu | Method set |
|------|------------|
| `T` | Mọi method với receiver `T` |
| `*T` | Mọi method với receiver `T` **hoặc** `*T` |

Hệ quả:

```go
type S struct{}

func (s S) V()  {}
func (s *S) P() {}

var x S
x.V()
x.P() // OK: Go lấy địa chỉ x (x addressable)

var p *S = &x
p.V() // OK: gọi trên bản sao *p
p.P()

// Interface
type VP interface {
    V()
    P()
}

var _ VP = &S{} // OK: *S có cả V và P
// var _ VP = S{} // LỖI: S không có P trong method set
```

Khi gán vào interface, compiler kiểm tra method set của **kiểu động** — không có “auto address” như lúc gọi trên biến addressable.

---

## 4. Embedding & method promotion

```go
type Reader struct{ /* ... */ }
func (r *Reader) Read(p []byte) (int, error) { /* ... */ }

type ReadWriter struct {
    *Reader // embedding
    *Writer
}

// ReadWriter có method Read, Write được promote
```

- Tên field ẩn = tên kiểu.
- Method của kiểu nhúng trở thành method của kiểu ngoài (nếu không bị che).
- Selector phân giải: field/method ngoài → depth embedding → báo lỗi nếu mơ hồ.

```go
type A struct{}
func (A) Hello() { fmt.Println("A") }

type B struct{}
func (B) Hello() { fmt.Println("B") }

type C struct {
    A
    B
}

// c.Hello() — lỗi ambiguous; phải c.A.Hello()
```

Embedding **không phải** kế thừa OOP: không có polymorphism subtype tự động ngoài việc thỏa interface.

---

## 5. Interface — thỏa mãn ngầm

```go
type Stringer interface {
    String() string
}

type User struct {
    Name string
}

func (u User) String() string {
    return u.Name
}

// User thỏa Stringer mà không cần ghi "implements"
var s fmt.Stringer = User{Name: "Ada"}
```

Định nghĩa interface theo **hành vi nhỏ**:

```go
type Closer interface {
    Close() error
}
```

Go 1.18+ — interface cũng dùng làm **constraint** generic (có thể chứa type element):

```go
type Number interface {
    ~int | ~int64 | ~float64
}
```

Interface thông thường chỉ chứa method (và từ 1.18 có thể type set khi là constraint).

Compile-time assert:

```go
var _ http.Handler = (*API)(nil)
```

---

## 6. `any` / empty interface

```go
type any = interface{} // alias chuẩn từ Go 1.18
```

Mọi kiểu thỏa `any` vì không yêu cầu method:

```go
var v any
v = 1
v = "hi"
v = User{}
```

Dùng khi:

- Container dị thể (trước generic), `fmt`, reflection.
- JSON `map[string]any`.

Với Go 1.21+, ưu tiên **generic** thay vì `any` + type assert khi kiểu biết được tại compile time.

---

## 7. Type assertion

```go
var s fmt.Stringer = User{Name: "Lin"}

u := s.(User) // panic nếu sai kiểu
u, ok := s.(User)
if !ok {
    // không phải User
}
```

Pointer vs value:

```go
var s fmt.Stringer = &User{Name: "Lin"}
_, ok1 := s.(User)  // false
_, ok2 := s.(*User) // true
```

Type switch — xem [statements.md](statements.md).

---

## 8. Interface value: (type, value)

Một biến interface lưu cặp **(dynamic type, dynamic value)**:

```go
var r io.Reader
fmt.Printf("%T %v\n", r, r) // <nil> <nil> — cả type và value nil

r = strings.NewReader("abc")
// type = *strings.Reader, value = pointer cụ thể
```

Gọi method trên interface → dispatch theo dynamic type.

So sánh interface:

- Hai interface equal nếu dynamic type giống nhau và dynamic value deep-equal (comparable).
- So với `nil`: chỉ true khi **cả** type và value là nil.

---

## 9. Nil interface vs nil pointer

Bẫy kinh điển:

```go
func returnsError() error {
    var p *MyError = nil
    return p // KHÔNG phải nil error!
}

type MyError struct{}

func (e *MyError) Error() string { return "my" }

err := returnsError()
fmt.Println(err == nil) // false — interface có type *MyError, value nil
```

Đúng:

```go
func returnsError() error {
    var p *MyError = nil
    if p == nil {
        return nil // interface thật sự nil
    }
    return p
}
```

Tương tự với bất kỳ interface nào: `var r io.Reader = (*os.File)(nil)` thì `r != nil`.

Khi implement `error` / trả interface: return `nil` tường minh, đừng return biến typed-nil.

---

## 10. Tránh interface pollution

**Interface pollution:** định nghĩa interface lớn / sớm / phía implementer thay vì phía consumer.

Khuyến nghị (Go proverb: *“The bigger the interface, the weaker the abstraction.”*):

| Nên | Tránh |
|-----|--------|
| Interface nhỏ (1–3 method) nơi **dùng** | Interface khổng lồ “IUserService” copy class |
| Accept interface, return concrete | Export interface chỉ có một implement trong package |
| Sinh interface từ nhu cầu test/mock | Đặt interface “phòng hờ” trước khi có consumer thứ hai |

```go
// Tốt: hàm nhận đúng thứ cần
func Copy(w io.Writer, r io.Reader) (int64, error) {
    return io.Copy(w, r)
}

// Package bạn: trả *File concrete
func Open(name string) (*File, error)

// Consumer khác tự mô tả interface nhỏ nếu cần mock
type Opener interface {
    Open(name string) (*File, error)
}
```

Không cần interface nếu chỉ có một implementation và không test-seam — dùng concrete.

---

## 11. Mẫu `io.Reader`

Chuẩn thư viện:

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}
```

Hợp đồng quan trọng (tóm tắt):

- Đọc vào `p`, trả số byte và lỗi.
- `err == io.EOF` khi hết dữ liệu; có thể `n > 0` kèm EOF.
- Không giữ `p` sau khi return.

Nhờ interface nhỏ, mọi nguồn (file, buffer, mạng, cipher) kết hợp được:

```go
r := io.LimitReader(file, 1024)
r = bufio.NewReader(r)
data, err := io.ReadAll(r)
```

Tương tự: `Writer`, `Closer`, `ReadCloser`, `Seeker` — compose thay vì god-object.

Viết kiểu thỏa `io.Reader`:

```go
type upperReader struct{ r io.Reader }

func (u upperReader) Read(p []byte) (int, error) {
    n, err := u.r.Read(p)
    for i := 0; i < n; i++ {
        if p[i] >= 'a' && p[i] <= 'z' {
            p[i] -= 'a' - 'A'
        }
    }
    return n, err
}
```

---

## 12. `comparable` & constraint (Go 1.18 / 1.20+)

### 12.1 `comparable`

Predeclared constraint: mọi kiểu có thể dùng làm operand của `==` / `!=` (và làm map key).

```go
func Keys[M ~map[K]V, K comparable, V any](m M) []K {
    keys := make([]K, 0, len(m))
    for k := range m {
        keys = append(keys, k)
    }
    return keys
}
```

### 12.2 Go 1.20+ — interface comparable hơn

Trước 1.20, interface không always comparable trong generic constraint theo cách mong muốn với một số tổ hợp. Từ **Go 1.20**:

- Giá trị interface comparable nếu dynamic type comparable.
- Constraint `comparable` chấp nhận kiểu có field interface (với hạn chế runtime: so sánh panic nếu dynamic value không comparable — ví dụ slice).

```go
type Pair struct {
    A any
    B any
}

// Pair có thể dùng với comparable trong một số ngữ cảnh 1.20+,
// nhưng Pair{A: []int{1}} == x có thể panic lúc chạy.
```

Thực tế: với generic map/set, giữ `K comparable` và tránh nhét slice/map/func vào key qua `any`.

### 12.3 Type set & approximation

```go
type Ordered interface {
    ~int | ~int64 | ~float64 | ~string
}

func Max[T Ordered](a, b T) T {
    if a > b {
        return a
    }
    return b
}
```

`~T` = underlying type là `T` (nhận cả `type MyInt int`).

---

## 13. Best practices

1. **Pointer receiver** khi mutate hoặc struct lớn; nhất quán trên toàn API kiểu đó.  
2. Nhớ **method set** khi gán vào interface — thường lưu `*T`.  
3. Interface **nhỏ**, định nghĩa phía **consumer**.  
4. Cẩn thận **typed nil** khi return `error` / interface.  
5. Compose giống `io`: nhiều interface hẹp hơn một interface “làm hết”.  
6. Generic + `comparable` / type set thay cho `any` + assert khi có thể (Go 1.21+).  
7. Embedding để promote method — không giả lập class hierarchy sâu.

### Ví dụ tổng hợp

```go
package store

import "io"

type Blob struct {
    key string
    r   io.Reader
}

func (b *Blob) Key() string { return b.key }

func (b *Blob) WriteTo(w io.Writer) (int64, error) {
    if b == nil || b.r == nil {
        return 0, io.ErrUnexpectedEOF
    }
    return io.Copy(w, b.r)
}

// Consumer định nghĩa đúng thứ cần:
type Keyed interface {
    Key() string
}

func CacheKey(k Keyed) string {
    return "blob:" + k.Key()
}
```

---

## Tài liệu liên quan

- [functions.md](functions.md) — hàm, closure, first-class
- [statements.md](statements.md) — type switch, scope
- [keywords.md](keywords.md) — `func`, `interface`, `type`, `struct`
