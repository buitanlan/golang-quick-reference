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
  - [6. Escape analysis (tổng quan)](#6-escape-analysis-tổng-quan)
  - [7. Receiver: pointer vs value](#7-receiver-pointer-vs-value)
  - [8. Slice, map, channel — “reference-like”](#8-slice-map-channel--reference-like)
  - [9. Pointer tới pointer](#9-pointer-tới-pointer)
  - [10. `unsafe.Pointer` (cảnh báo)](#10-unsafepointer-cảnh-báo)
  - [11. Pitfalls thường gặp](#11-pitfalls-thường-gặp)
  - [12. Best practices](#12-best-practices)

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

Không lấy địa chỉ của **map element**, **function return tạm** trong một số ngữ cảnh cũ — với composite literal thì được: `&User{}`.

```go
p := &User{Name: "An"} // OK
```

---

## 3. Zero value: `nil`

```go
var p *int
fmt.Println(p == nil) // true
// *p → panic: nil pointer dereference
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
- Cần arithmetic thật sự → `unsafe` (xem mục 10) — tránh trong code thường.

---

## 5. `new` vs `make`

| | `new(T)` | `make(T, ...)` |
|---|----------|----------------|
| Trả về | `*T` | `T` (slice/map/chan) |
| Dùng cho | mọi kiểu | chỉ slice, map, channel |
| Kết quả | pointer tới **zero value** | giá trị đã khởi tạo sẵn sàng dùng |

```go
p := new(int)     // *int, *p == 0
q := new(User)    // *User, zero fields

s := make([]int, 0, 10)     // slice
m := make(map[string]int)   // map
c := make(chan int, 1)      // channel
```

Tương đương thường dùng hơn `new`:

```go
p := new(User)
// gần với:
p2 := &User{}
```

> `make` **không** trả pointer. `make([]int, 3)` trả slice header (chứa pointer bên trong tới array).

---

## 6. Escape analysis (tổng quan)

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

## 7. Receiver: pointer vs value

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

## 8. Slice, map, channel — “reference-like”

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

## 9. Pointer tới pointer

Hiếm khi cần; chủ yếu API C / cập nhật biến pointer:

```go
func set(pp **int, v int) {
	*pp = &v // cẩn thận: v escape
}
```

Trong Go thuần, thường redesign API thay vì `**T`.

---

## 10. `unsafe.Pointer` (cảnh báo)

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

> **Khuyến nghị**: hầu hết codebase production **không** cần `unsafe`. Ưu tiên `encoding/binary`, `math`, slice tricks an toàn.

---

## 11. Pitfalls thường gặp

**1. Trả địa chỉ biến vòng lặp (trước Go 1.22):**

```go
var ptrs []*int
for i := 0; i < 3; i++ {
	ptrs = append(ptrs, &i) // trước 1.22: cả 3 trỏ cùng i
}
// Go 1.22+: mỗi iteration biến i mới — an toàn hơn
```

**2. Lấy địa chỉ range variable (cũ):** tương tự — copy ra biến local nếu nghi ngờ.

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

---

## 12. Best practices

1. Prefer giá trị nhỏ pass-by-value; pointer khi cần mutate / struct lớn / mutex.  
2. Document rõ method có sửa receiver không.  
3. Kiểm tra `nil` ở biên API công khai.  
4. Đừng dùng `unsafe` trừ khi bắt buộc.  
5. Nhớ slice/map/chan đã “chia sẻ dữ liệu” — không phải lúc nào cũng cần thêm `*`.  
6. Chạy `go test -race` khi có concurrency + shared pointer.  
7. Dùng `-gcflags="-m"` chỉ khi điều tra allocation.

---

### Cheat sheet

| Cú pháp | Ý nghĩa |
|---------|---------|
| `&v` | địa chỉ |
| `*p` | giá trị |
| `*T` | kiểu pointer |
| `nil` | zero pointer |
| `new(T)` | `*T` zero |
| `make` | slice/map/chan |
| `*T` receiver | mutate / lớn |
| `unsafe` | tránh |
