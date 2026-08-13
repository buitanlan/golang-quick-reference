# Pointer trong Go

Pointer lưu **địa chỉ** của biến. Go có pointer nhưng **không** có pointer arithmetic (trừ `unsafe`). Hiểu pointer là nền tảng để viết method hiệu quả, tránh copy lớn, và làm việc đúng với slice/map/channel.

---

## Mục lục

- [Pointer trong Go](#pointer-trong-go)
  - [Mục lục](#mục-lục)
  - [1. Tổng quan](#1-tổng-quan)
  - [2. Toán tử `&` và `*`](#2-toán-tử--và-)
  - [3. Zero value: `nil`](#3-zero-value-nil)
  - [4. Không có pointer arithmetic](#4-không-có-pointer-arithmetic)
  - [5. `new` vs `make`](#5-new-vs-make)
  - [6. Addressability (lấy được địa chỉ hay không)](#6-addressability-lấy-được-địa-chỉ-hay-không)
  - [7. Escape analysis (tổng quan)](#7-escape-analysis-tổng-quan)
  - [8. Receiver: pointer vs value](#8-receiver-pointer-vs-value)
  - [9. Slice, map, channel — “reference-like”](#9-slice-map-channel--reference-like)
  - [10. Pointer vào slice và lifetime của backing array](#10-pointer-vào-slice-và-lifetime-của-backing-array)
  - [11. Pointer tới pointer và pointer tới array](#11-pointer-tới-pointer-và-pointer-tới-array)
  - [12. So sánh pointer](#12-so-sánh-pointer)
  - [13. `unsafe.Pointer` (cảnh báo)](#13-unsafepointer-cảnh-báo)
  - [14. `runtime.Pinner` và cgo (Go 1.21+)](#14-runtimepinner-và-cgo-go-121)
  - [15. `weak.Pointer` và `runtime.AddCleanup` (Go 1.24+)](#15-weakpointer-và-runtimeaddcleanup-go-124)
  - [16. Pitfalls thường gặp](#16-pitfalls-thường-gặp)
  - [17. Khi nào KHÔNG dùng pointer](#17-khi-nào-không-dùng-pointer)
  - [18. Best practices](#18-best-practices)

---

## 1. Tổng quan

- Kiểu pointer: `*T` — trỏ tới giá trị kiểu `T`.  
- Zero value của `*T` là `nil`.  
- Dereference `nil` → **panic**.  
- GC quản lý bộ nhớ; không `free` thủ công.  
- Truyền pointer = truyền địa chỉ (rẻ), cho phép callee sửa giá trị gốc.

```go
x := 10
p := &x
*p = 20
fmt.Println(x) // 20
```

---

## 2. Toán tử `&` và `*`

| Toán tử | Nghĩa |
|---------|--------|
| `&v` | lấy địa chỉ của `v` → `*T` |
| `*p` | dereference — giá trị mà `p` trỏ tới |
| `*T` (trong kiểu) | kiểu “pointer tới T” |

```go
var n int = 42
var p *int = &n
fmt.Println(*p) // 42
*p = 100
fmt.Println(n)  // 100
```

Với struct, Go cho phép **sugar**:

```go
type User struct{ Name string }
u := &User{Name: "An"}
u.Name = "Bình" // tương đương (*u).Name
```

Không lấy địa chỉ của **map element** hay **giá trị trả về của hàm** — nhưng composite literal thì được (mục 6):

```go
p := &User{Name: "An"} // OK
```

---

## 3. Zero value: `nil`

```go
var p *int
fmt.Println(p == nil) // true
// *p → panic
```

Text panic thật khi deref pointer nil:

```text
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=... addr=0x0 pc=...]
```

Giá trị panic là `runtime.Error` (thường `runtime.errorString`), **không** phải `*runtime.TypeAssertionError`. Phân biệt trong `recover`:

```go
if re, ok := r.(runtime.Error); ok {
	// bug thật sự — nên log stack và cân nhắc re-panic
}
```

Kiểm tra trước khi dùng:

```go
func abs(p *int) int {
	if p == nil {
		return 0
	}
	if *p < 0 {
		return -*p
	}
	return *p
}
```

- Interface chứa typed nil khác `nil` interface — xem methods/interfaces.  
- `(*T)(nil)` vẫn là giá trị typed; method với pointer receiver có thể được gọi nếu không deref receiver (hiếm, dễ confuse).

```go
type T struct{}
func (t *T) Ok() { fmt.Println("ok") }
var p *T
p.Ok() // OK — không dùng field của t
```

---

## 4. Không có pointer arithmetic

```go
// C: p + 1
// Go: KHÔNG hợp lệ
p := &arr[0]
// p + 1  → compile error
```

- Duyệt mảng/slice bằng index hoặc `range`, không tăng pointer.  
- Cần arithmetic thật sự → `unsafe` (xem mục 13) — tránh trong code thường.

---

## 5. `new` vs `make`

| | `new(T)` / `new(expr)` | `make(T, ...)` |
|---|----------|----------------|
| Trả về | `*T` | `T` (slice/map/chan) |
| Dùng cho | mọi kiểu | chỉ slice, map, channel |
| Kết quả | pointer tới zero / tới bản copy của `expr` | giá trị đã khởi tạo sẵn sàng dùng |

```go
p := new(int)     // *int, *p == 0
q := new(User)    // *User, zero fields
r := new(42)      // Go 1.26+: *int, *r == 42 — không cần biến tạm

s := make([]int, 0, 10)     // slice
m := make(map[string]int)   // map
c := make(chan int, 1)      // channel
```

Tương đương thường dùng hơn `new(T)`:

```go
p := new(User)
// gần với:
p2 := &User{}
```

> `make` **không** trả pointer. `make([]int, 3)` trả slice header (chứa pointer bên trong tới array). Chi tiết `new(expr)`: [literals.md](literals.md) §13, [typesystem.md](typesystem.md) §5.

---

## 6. Addressability (lấy được địa chỉ hay không)

`&x` chỉ hợp lệ khi `x` **addressable**. Spec: biến, dereference pointer, slice index, addressable struct field, addressable array index, và composite literal (`&T{}`).

**Không** addressable (compiler từ chối `&`):

| Biểu thức | Lý do |
|-----------|--------|
| Map index `m[k]` | entry có thể di chuyển khi map grow |
| Kết quả hàm `f()` | temporary |
| Type conversion `T(v)` | temporary |
| Receive `<-ch` | temporary |
| Hằng / literal trần `42`, `"hi"` | không có ô nhớ (dùng `new(42)` từ 1.26) |

```go
m := map[string]int{"a": 1}
// p := &m["a"]          // LỖI: cannot take address of m["a"]
s := []int{1}
p := &s[0]               // OK — slice index addressable
u := User{Name: "An"}
q := &u.Name             // OK nếu u addressable
r := &User{Name: "An"}   // OK — composite literal
```

Hệ quả method: giá trị không addressable **không** gọi được pointer-receiver method — xem [methods-interfaces.md](methods-interfaces.md) §3.2.

---

## 7. Escape analysis (tổng quan)

Compiler quyết định biến sống trên **stack** hay **heap**:

- Nếu địa chỉ biến “thoát” ra ngoài hàm (return pointer, đóng vào goroutine/closure sống lâu, store vào heap structure…) → **escape** → heap.  
- Ngược lại có thể cấp phát trên stack — rẻ hơn, không áp lực GC.

```go
func f() *int {
	x := 1
	return &x // x escape → heap
}

func g() int {
	x := 1
	return x // x có thể ở stack
}
```

Xem quyết định:

```bash
go build -gcflags="-m" .
```

- Không tối ưu sớm theo “ép stack”; viết rõ ràng, để escape analysis làm việc.  
- Trả pointer tới large struct → thường heap; chấp nhận được nếu API cần.

---

## 8. Receiver: pointer vs value

```go
type Counter struct {
	n int
}

func (c Counter) Value() int { return c.n } // copy receiver
func (c *Counter) Inc()      { c.n++ }      // sửa gốc
```

**Khi dùng pointer receiver (`*T`)**:

- Method cần sửa receiver.  
- Struct lớn — tránh copy.  
- Có `sync.Mutex` field — **bắt buộc** không copy mutex → pointer.  
- Cần identity ổn định.

**Khi dùng value receiver (`T`)**:

- Kiểu nhỏ, bất biến.  
- Method thuần đọc, copy rẻ.  
- Muốn semantics “độc lập với bản gốc”.

Gọi method:

```go
var c Counter
c.Inc()   // Go lấy &c tự động nếu c addressable
p := &c
p.Value() // Go deref tự động
```

> Map element / non-addressable value **không** lấy được `&` → không gọi pointer method trực tiếp trên temporary.

Consistency: nếu đã có pointer method, thường dùng pointer cho cả bộ method set khi implement interface.

```go
type Inc interface{ Inc() }

var _ Inc = (*Counter)(nil) // *Counter implement Inc
// var _ Inc = Counter{}   // Counter (value) KHÔNG có method Inc trong method set
```

---

## 9. Slice, map, channel — “reference-like”

Chúng là **descriptor/header** chứa pointer tới dữ liệu nền:

```go
func appendOne(s []int) {
	s[0] = 99          // sửa shared array — thấy ở ngoài
	s = append(s, 1)   // có thể đổi header local — ngoài không thấy nếu realloc
}

a := []int{1, 2, 3}
appendOne(a)
fmt.Println(a[0]) // 99
```

| Kiểu | Copy khi gán/truyền | Dữ liệu nền |
|------|---------------------|-------------|
| slice | copy header (ptr,len,cap) | array dùng chung |
| map | copy handle | hashtable dùng chung |
| channel | copy handle | queue dùng chung |
| `*T` | copy địa chỉ | giá trị `T` dùng chung |
| array `[N]T` | copy **toàn bộ** | độc lập |
| struct | copy field | field pointer vẫn share |

```go
m := map[string]int{"a": 1}
func bump(m map[string]int) { m["a"]++ }
bump(m) // m["a"] == 2
```

Muốn callee thay **chính header** slice bên ngoài → truyền `*[]T`:

```go
func reset(s *[]int) { *s = nil }
```

---

## 10. Pointer vào slice và lifetime của backing array

Slice index addressable nên `&s[i]` lấy pointer vào **backing array**. Pointer đó giữ **toàn bộ** array sống — kể cả phần không còn trong slice:

```go
s := make([]byte, 1, 1<<20) // cap lớn
p := &s[0]
s = nil
// p vẫn neo 1 MiB backing array cho đến khi p biến mất
_ = p
```

- Sub-slice một buffer lớn rồi giữ pointer/slice nhỏ → **memory leak** kiểu Go (GC không thu phần “bỏ”). Copy ra slice mới (`slices.Clone` / `append([]T(nil), s...)`) khi cần tách lifetime.
- `append` có thể realloc → pointer cũ vào array **cũ** vẫn hợp lệ, nhưng không còn là phần tử của slice mới.
- Chi tiết aliasing: [collections.md](collections.md).

---

## 11. Pointer tới pointer và pointer tới array

Hiếm khi cần; chủ yếu API C / cập nhật biến pointer:

```go
func set(pp **int, v int) {
	*pp = &v // cẩn thận: v escape
}
```

Trong Go thuần, thường redesign API thay vì `**T`.

Pointer tới array `*[N]T` **không** phải slice. Slice có thể convert sang pointer-to-array (chia sẻ backing array):

```go
b := []byte("hello")
pa := (*[2]byte)(b)  // Go 1.17+: panic nếu len(b) < 2
arr := [5]byte(b)    // Go 1.20+: copy vào array; panic nếu len(b) < 5
_ = pa
_ = arr
```

- `*[N]T` hữu ích khi API C/syscall đòi kích thước cố định.
- Gán/truyền `*[N]T` copy con trỏ; gán `[N]T` copy **toàn bộ** N phần tử.
- Xem conversion: [typesystem.md](typesystem.md) §6.1.

---

## 12. So sánh pointer

- Hai pointer `==` khi cùng địa chỉ, hoặc cả hai `nil`.
- `*T` comparable nếu `T` comparable — nhưng so sánh pointer **không** so sánh giá trị `T`.
- Dereference rồi so: `*p == *q` (cần cả hai non-nil).
- Pointer tới biến khác nhau không bằng nhau dù giá trị giống:

```go
a, b := 1, 1
fmt.Println(&a == &b) // false
fmt.Println(*&a == *&b) // true
```

- `unsafe.Pointer` comparable; `uintptr` là số, không giữ object sống.

---

## 13. `unsafe.Pointer` (cảnh báo)

Package `unsafe` phá type safety:

```go
import "unsafe"

x := uint32(0x01020304)
p := unsafe.Pointer(&x)
b := (*[4]byte)(p) // reinterpret — phụ thuộc endian
```

**Nguy hiểm:**

- Không tương thích đảm bảo giữa phiên bản Go.  
- Dễ UB nếu alignment/lifetime sai.  
- `go vet` / checker có thể cảnh báo.  
- Chỉ dùng khi đo benchmark chứng minh cần, hoặc interop rõ ràng.

Chuyển đổi hợp lệ (theo docs):

- `*T` ↔ `unsafe.Pointer`  
- `unsafe.Pointer` ↔ `uintptr` (cẩn thận: `uintptr` không giữ object sống với GC)

```go
// Anti-pattern: lưu uintptr rồi dùng lại sau — object có thể bị GC
```

> **Khuyến nghị**: hầu hết codebase production **không** cần `unsafe`. Ưu tiên `encoding/binary`, `math`, slice tricks an toàn. API zero-copy `unsafe.String` / `SliceData`: [typesystem.md](typesystem.md) §16.

---

## 14. `runtime.Pinner` và cgo (Go 1.21+)

GC của Go **không di chuyển** object người dùng hiện tại, nhưng cgo vẫn cần **pin** khi C giữ pointer Go sau khi lời gọi cgo trả về, hoặc khi C đọc pointer nằm *bên trong* struct đã truyền sang C.

```go
var pin runtime.Pinner
defer pin.Unpin()

pin.Pin(goPtr)
// gọi C; C có thể giữ goPtr trong lúc Pinner còn sống
```

- `Unpin` gỡ **mọi** object đã pin của Pinner đó.
- Pinner unreachable → runtime tự unpin (tránh leak pin), nhưng phải **giữ Pinner sống** suốt thời gian C còn dùng pointer (`runtime.KeepAlive` nếu cần).
- Không pin = C có thể thấy pointer vào object đã bị GC.

---

## 15. `weak.Pointer` và `runtime.AddCleanup` (Go 1.24+)

`weak.Pointer[T]` (package `weak`) **không** giữ object sống. Khi chỉ còn weak ref, GC thu hồi; `Value()` trả `nil`.

```go
import "weak"

p := weak.Make(obj) // obj *T
if v := p.Value(); v != nil {
	use(v)
}
```

- Hai `weak.Pointer` `==` khi pointer gốc `==`, kể cả sau khi object chết (cặp “cùng nguồn”).
- Dùng cho cache / canonical map — không thay `*T` thường.

`runtime.AddCleanup` thay `SetFinalizer` khi cần lưới an toàn cho resource:

```go
type File struct{ fd int }

func open(fd int) *File {
	f := &File{fd: fd}
	runtime.AddCleanup(f, func(fd int) { syscall.Close(fd) }, f.fd)
	return f
}
```

- Cleanup nhận **arg copy**, không được capture `f` — nếu không object không bao giờ unreachable.
- Vẫn cần `Close()` + `defer` tường minh; cleanup chỉ là backup. Chi tiết: [typesystem.md](typesystem.md) §15.

---

## 16. Pitfalls thường gặp

**1. Trả địa chỉ biến vòng lặp (trước Go 1.22):**

```go
var ptrs []*int
for i := 0; i < 3; i++ {
	ptrs = append(ptrs, &i) // trước 1.22: cả 3 trỏ cùng i
}
// Go 1.22+: mỗi iteration biến i mới — an toàn hơn
// Luật theo `go` directive của file, không theo toolchain đang cài.
```

**2. Lấy địa chỉ range variable (cũ):** tương tự — copy ra biến local nếu file còn `go 1.21` trở xuống.

**3. Copy struct chứa Mutex:**

```go
type S struct {
	mu sync.Mutex
	n  int
}
a := S{}
b := a // SAO CHÉP mutex — BUG
```

Dùng pointer hoặc không export giá trị chứa mutex.

**4. Sửa qua value receiver rồi bất ngờ:**

```go
func (c Counter) IncWrong() { c.n++ } // sửa bản copy
```

**5. Nil map write:** `var m map[K]V; m[k]=v` → panic. Cần `make`.

**6. Giữ `&s[0]` sau khi thu hẹp slice** → neo backing array lớn (mục 10).

---

## 17. Khi nào KHÔNG dùng pointer

1. Kiểu nhỏ bất biến (`int`, `string`, `time.Time` nhỏ, struct vài word) — copy rẻ hơn chase pointer + GC.
2. Không cần mutate và không implement interface đòi pointer receiver.
3. Hot path đã escape vì boxing `any` / closure — thêm `*` không giúp.
4. Map value: `m[k].Inc()` không addressable; dùng `map[K]*T` hoặc copy-modify-write.
5. “Cho chắc” truyền `*T` mọi chỗ → tăng nil-check và allocation.

---

## 18. Best practices

1. Prefer giá trị nhỏ pass-by-value; pointer khi cần mutate / struct lớn / mutex.  
2. Document rõ method có sửa receiver không.  
3. Kiểm tra `nil` ở biên API công khai.  
4. Đừng dùng `unsafe` trừ khi bắt buộc.  
5. Nhớ slice/map/chan đã “chia sẻ dữ liệu” — không phải lúc nào cũng cần thêm `*`.  
6. Chạy `go test -race` khi có concurrency + shared pointer.  
7. Dùng `-gcflags="-m"` chỉ khi điều tra allocation.  
8. Resource: `Close()` + `defer`; `AddCleanup` chỉ là lưới an toàn.  
9. cgo giữ pointer Go → `runtime.Pinner`.

---

### Cheat sheet

| Cú pháp | Ý nghĩa |
|---------|---------|
| `&v` | địa chỉ (cần addressable) |
| `*p` | giá trị |
| `*T` | kiểu pointer |
| `nil` | zero pointer |
| `new(T)` | `*T` zero |
| `new(expr)` | `*T` trỏ tới bản copy (1.26) |
| `make` | slice/map/chan |
| `*T` receiver | mutate / lớn |
| `weak.Make` | weak pointer (1.24) |
| `runtime.Pinner` | pin cho cgo (1.21) |
| `unsafe` | tránh |
