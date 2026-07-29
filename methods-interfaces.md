# Method & Interface trong Go

Method gắn hành vi vào kiểu; interface mô tả tập method mà kiểu thỏa **ngầm**. Tài liệu nhắm Go 1.18–1.26 — chi tiết type system xem [typesystem.md](typesystem.md), generics xem [generics.md](generics.md).

---

## Mục lục

1. [Method — khái niệm](#1-method--khái-niệm)
2. [Receiver: value vs pointer](#2-receiver-value-vs-pointer)
3. [Method set (`T` vs `*T`) & addressability](#3-method-set-t-vs-t--addressability)
4. [Embedding & method promotion](#4-embedding--method-promotion)
5. [Interface — thỏa mãn ngầm](#5-interface--thỏa-mãn-ngầm)
6. [`any` / empty interface](#6-any--empty-interface)
7. [Type assertion](#7-type-assertion)
8. [Interface value: (type, value)](#8-interface-value-type-value)
9. [Nil interface vs nil pointer](#9-nil-interface-vs-nil-pointer)
10. [Tránh interface pollution](#10-tránh-interface-pollution)
11. [Mẫu `io.Reader`](#11-mẫu-ioreader)
12. [`comparable` & constraint (Go 1.18 / 1.20+)](#12-comparable--constraint-go-118--120)
13. [Chi phí dispatch & assert biên dịch](#13-chi-phí-dispatch--assert-biên-dịch)
14. [Best practices](#14-best-practices)

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
c.Inc()          // Go tự lấy &c nếu c addressable
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

Không có inheritance class kiểu C#/Java — tái sử dụng qua **embedding** ([structs-composition.md](structs-composition.md)) và **interface**.

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

Nếu **một** method của kiểu dùng pointer receiver, thường để **mọi** method mutating/API chính dùng pointer — tránh lẫn lộn method set (mục 3).

---

## 3. Method set (`T` vs `*T`) & addressability

**Method set** quyết định kiểu có thỏa interface nào và gọi method nào được.

| Kiểu | Method set |
|------|------------|
| `T` | Mọi method với receiver `T` |
| `*T` | Mọi method với receiver `T` **hoặc** `*T` |

Hệ quả khi gán vào interface / kiểm tra thỏa:

```go
type S struct{}

func (s S) V()  {}
func (s *S) P() {}

type VP interface {
    V()
    P()
}

var _ VP = (*S)(nil) // OK: *S có cả V và P
// var _ VP = S{}    // LỖI: method set của S không có P
```

### 3.1 Gọi trên biến vs gán vào interface

Trên biến **addressable**, compiler có thể lấy địa chỉ để gọi pointer method:

```go
var x S
x.V()
x.P() // OK: tương đương (&x).P()

var p *S = &x
p.V() // OK: gọi trên bản sao *p
p.P()
```

Khi gán vào interface, **không** có “auto address” — chỉ method set của kiểu động được xét.

### 3.2 Addressability

Không phải mọi giá trị kiểu `T` đều lấy `&` được. Giá trị không addressable (kết quả hàm, map index, literal tạm một số ngữ cảnh) **không** gọi được pointer method dù `*T` có method đó:

```go
func makeS() S { return S{} }

// makeS().P() // LỖI: cannot call pointer method P on S
s := makeS()
s.P() // OK — biến s addressable
```

Map index cũng không addressable:

```go
m := map[string]S{"a": {}}
// m["a"].P() // LỖI
tmp := m["a"]
tmp.P()
m["a"] = tmp
```

Quy tắc thực dụng: API public thường dùng pointer receiver + lưu/truyền `*T` khi cần thỏa interface có method mutate.

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
- Selector phân giải: field/method ngoài → depth embedding nhỏ hơn → **ambiguous** nếu nhiều ứng viên cùng depth.

```go
type A struct{}
func (A) Hello() { fmt.Println("A") }

type B struct{}
func (B) Hello() { fmt.Println("B") }

type C struct {
    A
    B
}

// c.Hello() — lỗi ambiguous; phải c.A.Hello() hoặc c.B.Hello()
```

Outer cùng tên **che** method được promote (không phải virtual override):

```go
type Truck struct {
    Vehicle
    Capacity int
}

func (t Truck) Move() { /* che Vehicle.Move khi gọi trên Truck */ }
```

Embedding **không phải** kế thừa OOP: không có polymorphism subtype tự động ngoài việc thỏa interface. Chi tiết field/tag/layout: [structs-composition.md](structs-composition.md).

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

Interface dùng như kiểu thường chỉ chứa method; type set (union/`~`) chủ yếu cho constraint — xem [generics.md](generics.md).

### 5.1 Compile-time assert

```go
var _ http.Handler = (*API)(nil) // *API phải có ServeHTTP
var _ fmt.Stringer = User{}      // value receiver OK với Stringer
```

Dạng `(*T)(nil)` phổ biến: không cấp phát, vẫn kiểm tra method set của `*T`.

### 5.2 Interface embedding — method trùng (Go 1.14+)

Embed nhiều interface được. Từ **Go 1.14**, method **trùng tên + cùng signature** từ các interface nhúng là hợp lệ (gộp một method):

```go
type ReadCloser interface {
    io.Reader
    io.Closer
}

type A interface{ Close() error }
type B interface{ Close() error }
type AB interface {
    A
    B // OK từ 1.14: Close trùng khớp
}
```

Signature **khác** → lỗi biên dịch (`duplicate method`):

```go
type Bad interface {
    interface{ Close() error }
    interface{ Close() string } // LỖI
}
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

Với Go **1.18–1.26**, khi kiểu biết được tại compile time hãy ưu tiên **generic** (`slices`, `maps`, `cmp`, …) thay vì `any` + type assert. `any` vẫn đúng chỗ cho biên động (JSON, plugin, `fmt`).

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

Với cây `error` đã wrap, đừng assert mù: dùng `errors.As` / `errors.AsType` (Go 1.26+) — xem [errors.md](errors.md).

Type switch — xem [statements.md](statements.md).

---

## 8. Interface value: (type, value)

Một biến interface lưu cặp **(dynamic type, dynamic value)** (chi tiết layout: [typesystem.md](typesystem.md) §11):

```go
var r io.Reader
fmt.Printf("%T %v\n", r, r) // <nil> <nil> — cả type và value nil

r = strings.NewReader("abc")
// type = *strings.Reader, value = pointer cụ thể
```

Gọi method trên interface → dispatch theo dynamic type (itab / dict — mục 13).

So sánh interface:

- Hai interface equal nếu dynamic type giống nhau và dynamic value so sánh được bằng `==`.
- So với `nil`: chỉ true khi **cả** type và value là nil.
- Dynamic value không comparable (slice/map/func) → **panic** lúc runtime khi so `==`.

---

## 9. Nil interface vs nil pointer

Bẫy kinh điển (**typed nil**):

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

Khi implement `error` / trả interface: return `nil` tường minh, đừng return biến typed-nil. Cùng chủ đề trong [errors.md](errors.md) §2.

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

**Accept interfaces, return structs:** tham số hàm/API nhận interface nhỏ; giá trị trả về thường là kiểu concrete (`*T`, struct) để caller còn gọi method ngoài interface và tránh mất thông tin kiểu.

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

### 12.2 Go 1.20 — interface thỏa `comparable`

`comparable` có từ **Go 1.18**. Thay đổi quan trọng ở **Go 1.20**: kiểu **interface** *thỏa mãn* (satisfies) constraint `comparable`, dù so sánh interface có thể **panic** ở runtime nếu dynamic value không comparable.

```go
// Go 1.18/1.19: Cache[any] lỗi. Go 1.20+: OK
type Cache[K comparable] map[K]int
var c = Cache[any]{}
c[42] = 1
// c[[]int{1}] = 3 // biên dịch OK, panic khi hash/so sánh key non-comparable
```

Hệ quả:

- Struct có field `any` / interface có thể dùng với `comparable` trong generic — trách nhiệm tránh panic thuộc về bạn.
- Với map/set generic, giữ `K` strictly comparable khi có thể; tránh nhét slice/map/func qua `any`.

Chi tiết “strictly comparable” vs spec-comparable, NaN: [typesystem.md](typesystem.md) §9.

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

`~T` = underlying type là `T` (nhận cả `type MyInt int`). Thực tế thường dùng `cmp.Ordered` (Go 1.21+) thay constraint tự chế — [generics.md](generics.md).

---

## 13. Chi phí dispatch & assert biên dịch

Gọi method qua interface không “miễn phí” hoàn toàn:

- Mỗi concrete type thỏa interface có **itab** (pair interface type ↔ concrete type) — cache trong runtime.
- Gọi method = gián tiếp qua itab (không phải monomorph hóa như generic instantiated).
- Hot path cực đoan: đo bằng benchmark; đôi khi generic hoặc concrete type nhanh hơn vì inline/devirtualize được.
- Box giá trị lớn vào interface có thể **copy + escape** sang heap.

Assert biên dịch (nhắc lại):

```go
var _ io.Writer = (*bytes.Buffer)(nil)
```

Lấy lỗi theo kiểu từ cây wrap (Go 1.26+):

```go
if pe, ok := errors.AsType[*fs.PathError](err); ok {
    fmt.Println(pe.Path)
}
```

---

## 14. Best practices

1. **Pointer receiver** khi mutate hoặc struct lớn; nhất quán trên toàn API kiểu đó.
2. Nhớ **method set** + **addressability** khi gán vào interface — thường lưu `*T`.
3. Interface **nhỏ**, định nghĩa phía **consumer**; accept interface, return concrete.
4. Cẩn thận **typed nil** khi return `error` / interface.
5. Compose giống `io`: nhiều interface hẹp hơn một interface “làm hết”.
6. Generic + `comparable` / type set / stdlib (`slices`, `maps`, `cmp`) thay `any` + assert khi kiểu tĩnh (Go 1.18–1.26).
7. Embedding để promote method — không giả lập class hierarchy sâu.
8. Lỗi typed: ưu tiên `errors.AsType` (1.26+) thay assert trực tiếp trên wrapped error.

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

- [structs-composition.md](structs-composition.md) — embedding, tag, layout
- [generics.md](generics.md) — type set, `comparable`, constraint
- [typesystem.md](typesystem.md) — comparable, interface runtime, `new(expr)`
- [functions.md](functions.md) — hàm, closure, first-class
- [statements.md](statements.md) — type switch, scope
- [errors.md](errors.md) — typed nil, `As` / `AsType`
- [keywords.md](keywords.md) — `func`, `interface`, `type`, `struct`
