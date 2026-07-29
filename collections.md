# Array, Slice & Map trong Go

Ba kiểu tập hợp cốt lõi: **array** (cố định), **slice** (view động trên array), **map** (hash table). Slice và map là công cụ hàng ngày; hiểu header (ptr/len/cap) tránh bug chia sẻ bộ nhớ và reallocation.

---

## Mục lục

- [Array, Slice \& Map trong Go](#array-slice--map-trong-go)
  - [Mục lục](#mục-lục)
  - [1. Array — kích thước cố định](#1-array--kích-thước-cố-định)
  - [2. Slice — header: ptr, len, cap](#2-slice--header-ptr-len-cap)
  - [3. `make`, `append`, `copy`](#3-make-append-copy)
  - [4. Slicing tricks](#4-slicing-tricks)
  - [5. Nil vs empty slice](#5-nil-vs-empty-slice)
  - [6. Map](#6-map)
  - [7. `delete`, comma-ok, `range`](#7-delete-comma-ok-range)
  - [8. Thứ tự duyệt map ngẫu nhiên](#8-thứ-tự-duyệt-map-ngẫu-nhiên)
  - [9. `clear` (Go 1.21+)](#9-clear-go-121)
  - [10. Package `slices` \& `maps` (Go 1.21+)](#10-package-slices--maps-go-121)
  - [11. `sync.Map` (tóm tắt)](#11-syncmap-tóm-tắt)
  - [12. Pitfalls \& best practices](#12-pitfalls--best-practices)

---

## 1. Array — kích thước cố định

```go
var a [3]int           // [0,0,0]
b := [3]int{1, 2, 3}
c := [...]int{1, 2, 3} // độ dài suy ra = 3
```

- Kiểu gồm luôn độ dài: `[3]int` ≠ `[4]int`.  
- Gán array = **copy toàn bộ**.  
- Ít dùng trực tiếp; thường embed trong struct hoặc làm backing của slice.  
- Comparable nếu element comparable.

```go
x := [2]int{1, 2}
y := x
y[0] = 9
fmt.Println(x[0]) // 1 — độc lập
```

Truyền vào hàm copy cả array → với array lớn nên dùng slice hoặc `*[N]T`.

---

## 2. Slice — header: ptr, len, cap

Slice = descriptor:

```text
+--------+--------+--------+
|  ptr   |  len   |  cap   |
+--------+--------+--------+
    |
    v
 [ backing array ... ]
```

```go
s := []int{1, 2, 3, 4, 5}
fmt.Println(len(s), cap(s)) // 5 5
t := s[1:4]                 // ptr→2, len=3, cap=4 (tới cuối array gốc)
```

- `len`: số phần tử dùng được.  
- `cap`: số phần tử từ ptr tới hết backing.  
- Gán slice copy **header**, không copy element → chia sẻ array.

```go
a := []int{1, 2, 3}
b := a
b[0] = 99
fmt.Println(a[0]) // 99
```

---

## 3. `make`, `append`, `copy`

```go
s1 := make([]int, 5)     // len=5, cap=5, zero elements
s2 := make([]int, 0, 8)  // len=0, cap=8 — sẵn sàng append
s3 := make([]int, 5, 10) // len=5, cap=10
```

**append** — có thể realloc khi `len+n > cap`:

```go
s := make([]int, 0, 2)
s = append(s, 1, 2)
s = append(s, 3) // có thể đổi ptr → phải gán lại s
```

- **Luôn** dùng `s = append(s, ...)`.  
- Append nhiều: `append(s, t...)`.

**copy**:

```go
dst := make([]int, len(src))
n := copy(dst, src) // n = số phần tử copy (min len)
```

- `copy` không realloc; overlap được hỗ trợ.  
- Clone độc lập: `append([]T(nil), src...)` hoặc `slices.Clone` (1.21+).

---

## 4. Slicing tricks

```go
s := []int{0, 1, 2, 3, 4, 5}
s[low:high]      // len = high-low; cap = cap(s)-low
s[low:high:max]  // full slice: cap = max-low (giới hạn cap!)
```

Giới hạn cap để `append` không ghi đè phần còn lại của array gốc:

```go
head := s[:2:2]
head = append(head, 99) // cấp phát mới, không đè s[2]
```

Các idiom:

```go
s = s[:0]              // giữ cap, xóa logic
s = s[:len(s)-1]       // pop cuối (cẩn thận len=0)
s = append(s[:i], s[i+1:]...) // xóa index i
s = append(s[:i], s[i+n:]...) // xóa n phần tử từ i
```

Reverse (in-place):

```go
for i, j := 0, len(s)-1; i < j; i, j = i+1, j-1 {
	s[i], s[j] = s[j], s[i]
}
```

---

## 5. Nil vs empty slice

```go
var nilSlice []int          // nil, len=0, cap=0
empty := []int{}            // non-nil, len=0
empty2 := make([]int, 0)    // non-nil, len=0
```

| | nil slice | empty non-nil |
|---|-----------|---------------|
| `s == nil` | true | false |
| `len(s)` | 0 | 0 |
| `append` | OK | OK |
| `range` | 0 lần | 0 lần |
| `json.Marshal` | `null` | `[]` |

- API thường chấp nhận cả hai như “không có phần tử”.  
- Muốn JSON luôn `[]` → khởi tạo `make([]T, 0)`.

---

## 6. Map

```go
var m map[string]int          // nil map
m = make(map[string]int)      // sẵn sàng ghi
m2 := map[string]int{"a": 1}  // literal
```

- Key phải **comparable** (không slice/map/function).  
- Value bất kỳ.  
- Đọc từ nil map → zero value (không panic).  
- Ghi vào nil map → **panic**.

```go
var m map[string]int
fmt.Println(m["x"]) // 0
// m["x"] = 1       // panic
```

Độ phức tạp trung bình O(1) get/put; worst-case kém hơn khi hash xấu / resize.

---

## 7. `delete`, comma-ok, `range`

```go
m := map[string]int{"a": 1, "b": 2}
delete(m, "a")     // xóa; không có key → no-op
delete(m, "missing")

v, ok := m["b"]
if !ok {
	// không có key — phân biệt value = zero vs thiếu key
}
```

Comma-ok bắt buộc khi zero value hợp lệ (0, "", false…).

```go
for k, v := range m {
	fmt.Println(k, v)
}
for k := range m {
	delete(m, k) // được phép xóa key đang duyệt (Go spec)
}
```

Chỉ value: `for _, v := range m`.

---

## 8. Thứ tự duyệt map ngẫu nhiên

```go
for k := range m { ... } // thứ tự không ổn định giữa các lần chạy
```

- Runtime cố ý randomize để tránh phụ thuộc thứ tự.  
- Cần ổn định: lấy keys, `sort.Strings(keys)`, rồi duyệt.  
- Go 1.21+: `slices.Sorted(maps.Keys(m))`.

```go
keys := make([]string, 0, len(m))
for k := range m {
	keys = append(keys, k)
}
sort.Strings(keys)
for _, k := range keys {
	fmt.Println(k, m[k])
}
```

---

## 9. `clear` (Go 1.21+)

Builtin `clear`:

```go
s := []int{1, 2, 3}
clear(s) // zero từng element; len/cap giữ nguyên

m := map[string]int{"a": 1}
clear(m) // xóa mọi entry; map vẫn usable
```

- Slice: không đổi len — khác `s = s[:0]`.  
- Map: tương đương xóa hết key.  
- Cũng dùng với pointer? Không — chỉ type có clear semantics (slice, map; và từ 1.21 docs).

---

## 10. Package `slices` & `maps` (Go 1.21+)

```go
import (
	"maps"
	"slices"
)
```

**slices** (một phần):

```go
slices.Contains(s, x)
slices.Index(s, x)
slices.Clone(s)
slices.Equal(a, b)
slices.Sort(s)           // ordered types
slices.SortFunc(s, cmp)
slices.Compact(s)        // bỏ trùng liên tiếp
slices.Delete(s, i, j)
slices.Insert(s, i, elems...)
slices.BinarySearch(s, x)
```

**maps**:

```go
maps.Clone(m)
maps.Copy(dst, src)
maps.Equal(m1, m2)
maps.Keys(m)    // iter (Go 1.23+ range over func; 1.21/1.22 có Keys trả slice tùy version)
maps.Values(m)
maps.DeleteFunc(m, func(k K, v V) bool { return v == 0 })
```

> Go 1.23 giới thiệu `range` over iterator; API `maps.Keys` trả về `iter.Seq`. Với 1.21/1.22 kiểm tra docs đúng phiên bản bạn dùng — pattern “collect keys rồi sort” vẫn portable.

Ví dụ clone slice an toàn:

```go
dst := slices.Clone(src) // độc lập backing
```

---

## 11. `sync.Map` (tóm tắt)

`sync.Map` — map concurrent-safe trong `sync`:

- Phù hợp: key ổn định, write-once / read-many, hoặc disjoint key goroutines.  
- API: `Store`, `Load`, `LoadOrStore`, `Delete`, `Range`.  
- **Không** phải thay thế mặc định cho `map` + `Mutex` — benchmark trước.

```go
var sm sync.Map
sm.Store("a", 1)
if v, ok := sm.Load("a"); ok {
	fmt.Println(v.(int))
}
```

Chi tiết concurrency: [concurrency.md](concurrency.md).

---

## 12. Pitfalls & best practices

1. Quên `s = append(s, ...)` → mất phần tử mới.  
2. Subslice giữ sống cả backing lớn → memory leak logic; copy khi giữ lâu.  
3. Append vào subslice đè dữ liệu “đuôi”. Dùng full slice `s[i:j:j]`.  
4. Map không an toàn cho concurrent read+write → mutex hoặc sync.Map.  
5. Không dựa vào thứ tự `range` map.  
6. Phân biệt nil/empty khi JSON.  
7. Preallocate `make([]T, 0, n)` / `make(map[K]V, n)` khi biết n.  
8. Array lớn trên stack có thể đắt — cân nhắc slice.

### Cheat sheet

| Thao tác | Cú pháp |
|----------|---------|
| Tạo slice | `make([]T, len, cap)` |
| Thêm | `s = append(s, x)` |
| Copy | `copy(dst, src)` / `slices.Clone` |
| Xóa map key | `delete(m, k)` |
| Có key? | `v, ok := m[k]` |
| Xóa hết | `clear(m)` / `clear(s)` (1.21) |
| Nil slice JSON | `null` vs `[]` |
