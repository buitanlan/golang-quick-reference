# Array, Slice & Map trong Go

Ba kiểu tập hợp cốt lõi: **array** (cố định), **slice** (view động trên array), **map** (hash table). Slice và map là công cụ hàng ngày; hiểu header (ptr/len/cap) tránh bug chia sẻ bộ nhớ và reallocation.

> Tài liệu nhắm **Go 1.26**; số liệu đo trên `go1.26.5 windows/amd64`. Mọi hàm `slices`/`maps` đều ghi version xuất hiện.

---

## Mục lục

- [Array, Slice \& Map trong Go](#array-slice--map-trong-go)
  - [Mục lục](#mục-lục)
  - [1. Array — kích thước cố định](#1-array--kích-thước-cố-định)
  - [2. Slice — header: ptr, len, cap](#2-slice--header-ptr-len-cap)
  - [3. `make`, `append`, `copy`](#3-make-append-copy)
    - [3.1 Chiến lược tăng cap của `append`](#31-chiến-lược-tăng-cap-của-append)
    - [3.2 `copy` chi tiết](#32-copy-chi-tiết)
  - [4. Slicing tricks](#4-slicing-tricks)
    - [4.1 Aliasing \& memory retention](#41-aliasing--memory-retention)
  - [5. Nil vs empty slice](#5-nil-vs-empty-slice)
  - [6. Map](#6-map)
    - [6.1 Bên trong map: Swiss table (Go 1.24+)](#61-bên-trong-map-swiss-table-go-124)
    - [6.2 Map element không addressable](#62-map-element-không-addressable)
  - [7. `delete`, comma-ok, `range`](#7-delete-comma-ok-range)
  - [8. Thứ tự duyệt map ngẫu nhiên](#8-thứ-tự-duyệt-map-ngẫu-nhiên)
  - [9. `clear` (Go 1.21+)](#9-clear-go-121)
  - [10. Package `slices` \& `maps`](#10-package-slices--maps)
    - [10.1 Bảng tra `slices` theo version](#101-bảng-tra-slices-theo-version)
    - [10.2 Bảng tra `maps` theo version](#102-bảng-tra-maps-theo-version)
    - [10.3 Bẫy: hàm ghi vào slice gốc](#103-bẫy-hàm-ghi-vào-slice-gốc)
  - [11. Iterator trên collection (Go 1.23+)](#11-iterator-trên-collection-go-123)
  - [12. `sync.Map` (tóm tắt)](#12-syncmap-tóm-tắt)
  - [13. Pitfalls \& best practices](#13-pitfalls--best-practices)

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

Array vs slice — khác biệt cần nhớ:

| | `[N]T` | `[]T` |
|---|---|---|
| Độ dài | phần của **kiểu**, hằng compile-time | runtime, đổi được |
| Gán / truyền tham số | copy giá trị | copy header (3 word) |
| `==` | Có, nếu `T` comparable | Không (chỉ so `nil`) |
| Map key | Có | Không |
| `len` là constant | Có: `const n = len(a)` | Không |
| Zero value | mọi phần tử zero, dùng ngay | `nil`, `append` được |

Chuyển đổi hai chiều (chi tiết ở [typesystem.md](typesystem.md)):

```go
s := []int{1, 2, 3, 4}
arr := [3]int(s[:3])   // Go 1.20+: copy, panic nếu len < 3
pa := (*[2]int)(s)     // Go 1.17+: KHÔNG copy — chia sẻ backing
sl := pa[:]            // quay lại slice trên cùng bộ nhớ
```

`*[N]T` là cách truyền array không copy mà vẫn giữ độ dài trong kiểu — hữu ích cho buffer cố định (`*[16]byte`).

---

## 2. Slice — header: ptr, len, cap

Slice = descriptor 3 word (24 byte trên 64-bit):

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
- `unsafe.Sizeof(s)` luôn 24 bất kể độ dài — đây là kích thước header, không phải dữ liệu.

```go
a := []int{1, 2, 3}
b := a
b[0] = 99
fmt.Println(a[0]) // 99
```

Điểm quan trọng: **slice không có “identity”**. Hai slice cùng trỏ vào một backing vẫn là hai giá trị độc lập; sửa `len`/`cap` của bản này không ảnh hưởng bản kia, nhưng sửa *element* thì có.

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
- Append string vào `[]byte`: `append(b, "abc"...)` — trường hợp đặc biệt của spec.

### 3.1 Chiến lược tăng cap của `append`

Runtime (`runtime.nextslicecap` trong go1.26.5) quyết định cap mới theo `newLen` và `oldCap`:

1. `newLen > 2*oldCap` → dùng luôn `newLen` (append nhiều phần tử một lần thì không over-allocate).
2. Ngược lại, `oldCap < 256` → **gấp đôi**.
3. `oldCap >= 256` → cộng dần `newcap += (newcap + 3*256) / 4` (≈ **1.25×**) cho tới khi đủ.

Sau đó số byte được **làm tròn lên size class** của allocator, nên cap thực tế thường không phải luỹ thừa 2.

Đo thật với `[]int` (escape ra heap), in `cap` mỗi lần nó đổi:

```text
1 2 4 8 16 32 64 128 256 512 848 1280 1792 2560 3408 5120
```

- Tới 512 là gấp đôi; từ đó `512 → 832` theo công thức 1.25×, làm tròn size class thành **848**.
- Với `[]string` (16 byte/phần tử) dãy đo được là `2 4 8 16 32 71 143 303` — các số lẻ chính là hệ quả của việc làm tròn size class, **đừng** hard-code chúng.

Một chi tiết dễ gây bối rối khi benchmark: nếu slice **không escape**, compiler cấp phát buffer trên stack và đo được cap lớn hơn mong đợi.

```go
func local() {
	var y []int
	y = append(y, 1)
	fmt.Println(cap(y)) // 4  — buffer stack (32 byte)
}

func escaping() []int {
	var y []int
	y = append(y, 1)
	return y            // cap(y) == 1 — cấp phát heap đúng size class nhỏ nhất
}
```

Hệ quả thực tế:

- Đừng suy ra “growth factor” từ một vòng lặp cục bộ trong `main`; kết quả phụ thuộc escape analysis.
- Biết trước số phần tử → `make([]T, 0, n)` để có **1 lần** cấp phát, hoặc `slices.Grow(s, n)`.
- Cần trả cap về đúng len (giải phóng phần dư trước khi giữ lâu) → `slices.Clip(s)`.

### 3.2 `copy` chi tiết

```go
dst := make([]int, len(src))
n := copy(dst, src) // n = min(len(dst), len(src))
```

- `copy` **không** realloc, không đổi `len(dst)`: `copy(make([]int, 0, 100), src)` copy **0** phần tử — lỗi kinh điển.
- Nguồn/đích được phép **overlap** (semantics `memmove`, không phải `memcpy`):

```go
c := []int{1, 2, 3, 4, 5}
copy(c[1:], c[:4])
fmt.Println(c) // [1 1 2 3 4] — dịch phải 1, không bị "smear"
```

- Đặc biệt: `copy([]byte, string)` hợp lệ (`copy(b, "hi")`).
- Clone độc lập: `slices.Clone(src)` (1.21+). `append([]T(nil), src...)` cũng được nhưng trả `nil` khi `src` rỗng.

---

## 4. Slicing tricks

```go
s := []int{0, 1, 2, 3, 4, 5}
s[low:high]      // len = high-low; cap = cap(s)-low
s[low:high:max]  // full slice: cap = max-low (giới hạn cap!)
```

- `0 <= low <= high <= max <= cap(s)` — vượt ra là **panic**, và `high` được phép vượt `len(s)` (miễn còn trong `cap`).
- Slicing **không bao giờ** copy: `s[a:b]` chỉ tạo header mới.
- `s[low:]`, `s[:high]`, `s[:]` là dạng rút gọn; `s[:]` trên array cho slice trên chính array đó (array phải addressable).

Giới hạn cap để `append` không ghi đè phần còn lại của array gốc:

```go
base := []int{0, 1, 2, 3, 4, 5}

head := base[:2:2]
head = append(head, 99) // cấp phát mới
// base = [0 1 2 3 4 5] — nguyên vẹn

head2 := base[:2]
head2 = append(head2, 99) // còn cap → ghi thẳng vào backing
// base = [0 1 99 3 4 5]  ← đã bị sửa
```

Đây là bug hay gặp nhất khi một hàm nhận slice rồi `append`: **caller có thể thấy phần tử sau `len` bị đổi**. Quy ước: hàm nào định `append` vào slice của người khác thì nhận `s[:len(s):len(s)]` hoặc `slices.Clone`.

Các idiom:

```go
s = s[:0]              // giữ cap, xóa logic
s = s[:len(s)-1]       // pop cuối (cẩn thận len=0)
s = append(s[:i], s[i+1:]...) // xóa index i (KHÔNG zero phần đuôi)
s = slices.Delete(s, i, i+1)  // 1.21+: giống trên nhưng zero phần đuôi
```

Reverse (in-place) — hoặc dùng `slices.Reverse` (1.21+):

```go
for i, j := 0, len(s)-1; i < j; i, j = i+1, j-1 {
	s[i], s[j] = s[j], s[i]
}
```

### 4.1 Aliasing & memory retention

Slice giữ **cả backing array** sống, không chỉ phần `[0:len]`:

```go
func firstLine(data []byte) []byte {
	i := bytes.IndexByte(data, '\n')
	return data[:i] // giữ nguyên 100MB backing nếu data lớn
}
```

- Trả về sub-slice của buffer lớn ⇒ GC không thu hồi được buffer. Muốn thả: `slices.Clone(data[:i])` hoặc `bytes.Clone`.
- Ngược lại, `s = s[1:]` (cắt đầu) làm `ptr` tiến lên nhưng phần bị bỏ vẫn nằm trong cùng allocation → vẫn sống.
- Slice của **pointer/interface**: `s = s[:0]` giữ nguyên các pointer trong `[0:cap]` → object bị trỏ tới không được thu hồi. Dùng `clear(s)` **trước** khi `s = s[:0]`:

```go
clear(s)   // zero mọi element trong [0:len] → thả reference
s = s[:0]  // reuse buffer
```

- `*[N]T` từ một slice lớn (`(*[2]int)(big)`) cũng giữ toàn bộ allocation — cùng cái bẫy.

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
| `len(s)` / `cap(s)` | 0 / 0 | 0 / 0 |
| `append` | OK | OK |
| `range` | 0 lần | 0 lần |
| `s[:0]`, `copy`, `clear` | OK | OK |
| `json.Marshal` | `null` | `[]` |
| `slices.Equal(nil, []int{})` | `true` — coi như bằng nhau | |

- API thường chấp nhận cả hai như “không có phần tử”.
- Muốn JSON luôn `[]` → khởi tạo `make([]T, 0)`.
- `slices` giữ **nilness**: `slices.Delete` trả kết quả rỗng vẫn `nil` nếu input `nil`; `slices.Concat` trả `nil` khi kết quả rỗng; `slices.Repeat` **không bao giờ** `nil`.
- Đừng viết `if s != nil` để kiểm tra “có dữ liệu” — dùng `len(s) > 0`.

---

## 6. Map

```go
var m map[string]int          // nil map
m = make(map[string]int)      // sẵn sàng ghi
m2 := map[string]int{"a": 1}  // literal
m3 := make(map[string]int, 1000) // hint số entry → ít rehash
```

- Key phải **comparable** (không slice/map/function).
- Value bất kỳ, kể cả map/slice/func.
- Đọc từ nil map → zero value (không panic). `len`, `range`, `delete` trên nil map cũng OK.
- Ghi vào nil map → **panic**.

```go
var m map[string]int
fmt.Println(len(m), m["x"]) // 0 0
for range m { }             // 0 lần
delete(m, "x")              // no-op
m["x"] = 1                  // panic: assignment to entry in nil map
```

Bẫy comparable + float: `NaN != NaN` nên mỗi lần ghi tạo entry mới và **không đọc lại được** (chi tiết ở [typesystem.md](typesystem.md) §9.2):

```go
nan := math.NaN()
m := map[float64]int{}
m[nan] = 1
m[nan] = 2
fmt.Println(len(m))  // 2 — hai entry "rác", delete(m, nan) không xóa được
clear(m)             // cách duy nhất để dọn
```

Độ phức tạp trung bình O(1) get/put; worst-case kém hơn khi hash xấu / resize.

### 6.1 Bên trong map: Swiss table (Go 1.24+)

Từ Go 1.24 map builtin được hiện thực lại theo **Swiss table** (thiết kế của Abseil) — trong go1.26.5 nằm ở `internal/runtime/maps`. Trước đó là bucket chaining với 8 slot/bucket + overflow bucket.

Điều **không** đổi (và bạn được phép dựa vào):

- Thứ tự duyệt vẫn không xác định (§8).
- Ghi vào nil map vẫn panic; concurrent read+write vẫn bị runtime phát hiện.
- Element vẫn không addressable (§6.2).
- Amortized O(1), key phải comparable.

Điều đổi và **không** nên dựa vào:

- Layout bộ nhớ, số entry mỗi lần grow, cap thực tế — không có `cap(map)` và chưa bao giờ có.
- Thứ tự duyệt cụ thể và “mức độ ngẫu nhiên” của nó (§8 đo lại trên chính toolchain của bạn).
- Số liệu hiệu năng: release notes Go 1.24 nói tổng hợp map + allocator + mutex giảm ~2–3% CPU overhead trên bộ benchmark của họ, **không** phải “map nhanh hơn 2 lần”. Muốn biết ảnh hưởng lên code của bạn thì benchmark.

`delete` không thu nhỏ map: bộ nhớ bucket được giữ lại để reuse. Muốn thực sự giải phóng, tạo map mới:

```go
m = make(map[K]V, len(keep)) // thay vì delete từng key rồi giữ map cũ
```

### 6.2 Map element không addressable

Map element không có địa chỉ ổn định (grow có thể di chuyển entry), nên compiler chặn:

```go
type P struct{ X int }
m := map[string]P{"a": {1}}

// _ = &m["a"]    // invalid operation: cannot take address of m["a"]
//                //   (map index expression of struct type P)
// m["a"].X = 2   // cannot assign to struct field m["a"].X in map
```

Ba cách xử lý:

```go
// 1. Read-modify-write toàn bộ value
p := m["a"]
p.X = 2
m["a"] = p

// 2. Dùng map[K]*V khi cần sửa in-place
mp := map[string]*P{"a": {1}}
mp["a"].X = 2

// 3. Value là slice/map (đã là reference) — append vẫn phải gán lại
ms := map[string][]int{}
ms["a"] = append(ms["a"], 1) // đọc nil, append, ghi lại: hợp lệ
```

Ngược lại **slice element addressable**: `&s[0]`, `s[0].X = 2` đều hợp lệ.

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
```

Sửa map trong lúc `range` — luật spec (`for` statements, range clause):

- Entry **chưa được duyệt tới** mà bị xóa → chắc chắn không xuất hiện.
- Entry được **tạo mới** trong lúc duyệt → *có thể* xuất hiện hoặc bị bỏ qua; mỗi entry và mỗi lần duyệt có thể khác nhau.
- Vì vậy `delete` khi duyệt là hợp lệ và có định nghĩa; `insert` khi duyệt là hợp lệ nhưng **kết quả không xác định**.

```go
for k := range m {
	delete(m, k) // OK — mọi key đều được duyệt đúng 1 lần rồi xóa
}

for k := range m {
	m[k+"!"] = 1 // hợp lệ nhưng vòng lặp có thể chạy bao nhiêu lần cũng được
}
```

Chỉ value: `for _, v := range m`. Xóa theo điều kiện: `maps.DeleteFunc(m, pred)` (1.21+).

---

## 8. Thứ tự duyệt map ngẫu nhiên

Spec: *“The iteration order over maps is not specified and is not guaranteed to be the same from one iteration to the next.”* Runtime cố ý randomize để code không lỡ phụ thuộc vào thứ tự.

```go
for k := range m { ... } // thứ tự không ổn định giữa các lần duyệt
```

**Randomize là điểm bắt đầu, không phải shuffle.** Đo trên go1.26.5 (1000 lần `range`):

| Map | Số thứ tự phân biệt quan sát được |
|---|---|
| 3 key (`a`,`b`,`c`) | 3 — đúng các phép quay `abc`, `bca`, `cab` |
| 40 key | 40 |

Nghĩa là runtime chọn offset bắt đầu ngẫu nhiên rồi đi theo layout nội bộ. Đừng dùng `range` map để lấy phần tử ngẫu nhiên hay để trộn dữ liệu — dùng `math/rand/v2` (`rand.Shuffle`, `rand.IntN`).

Cần thứ tự ổn định thì sort key. Từ 1.23 có one-liner:

```go
for _, k := range slices.Sorted(maps.Keys(m)) {
	fmt.Println(k, m[k])
}
```

Bản “thủ công” (portable mọi version, và nhanh hơn khi cần cả key lẫn value):

```go
keys := make([]string, 0, len(m))
for k := range m {
	keys = append(keys, k)
}
slices.Sort(keys) // hoặc sort.Strings(keys) trước 1.21
for _, k := range keys {
	fmt.Println(k, m[k])
}
```

---

## 9. `clear` (Go 1.21+)

Builtin `clear` có signature `clear[T ~[]Type | ~map[Type]Type1](t T)` — hoạt động khác nhau theo kiểu:

```go
s := []int{1, 2, 3}
clear(s) // zero element trong [0:len]; len/cap giữ nguyên → [0 0 0]

m := map[string]int{"a": 1}
clear(m) // xóa mọi entry; map vẫn usable
```

| | `clear(s)` slice | `clear(m)` map |
|---|---|---|
| Sau lệnh | `len` không đổi, element = zero | `len(m) == 0` |
| So với cách khác | khác `s = s[:0]` (đó chỉ đổi len) | mạnh hơn vòng `delete`: dọn cả key `NaN` |
| Tham số `nil` | no-op | no-op |
| Type parameter | được, nếu type set chỉ chứa slice/map | |

- Không dùng cho array, pointer, struct hay channel.
- Cặp `clear(s); s = s[:0]` là idiom reuse buffer mà không giữ reference (§4.1).
- `clear` là **builtin**, dùng được cả với type parameter — ví dụ chuẩn cho thấy union type set không cần “core type” chung ([generics.md](generics.md) §6).

---

## 10. Package `slices` & `maps`

```go
import (
	"maps"
	"slices"
)
```

Hai package này thay thế phần lớn `sort.Slice` và các helper viết tay. Chúng nhận constraint `~[]E` / `~map[K]V` nên hoạt động cả với named type (`type IDs []int`) và **trả về đúng named type đó**.

### 10.1 Bảng tra `slices` theo version

| Hàm | Version | Ghi chú |
|---|---|---|
| `Contains`, `Index`, `ContainsFunc`, `IndexFunc` | 1.21 | tìm tuyến tính |
| `Equal`, `EqualFunc`, `Compare`, `CompareFunc` | 1.21 | `Equal(nil, []T{})` → `true` |
| `Clone` | 1.21 | shallow copy, cap = len |
| `Sort`, `IsSorted` | 1.21 | `cmp.Ordered`, pdqsort |
| `SortFunc` | 1.21 | **không** đảm bảo stable |
| `SortStableFunc`, `IsSortedFunc` | 1.21 | `cmp func(a, b E) int` |
| `BinarySearch`, `BinarySearchFunc` | 1.21 | slice **phải** đã sort; trả `(pos, found)` |
| `Min`, `Max`, `MinFunc`, `MaxFunc` | 1.21 | panic nếu slice rỗng |
| `Compact`, `CompactFunc` | 1.21 | bỏ **trùng liên tiếp** — sort trước nếu muốn unique |
| `Delete`, `DeleteFunc` | 1.21 | zero phần đuôi |
| `Insert`, `Replace` | 1.21 | |
| `Grow`, `Clip` | 1.21 | `Grow(s, n)` đảm bảo thêm được n phần tử |
| `Reverse` | 1.21 | in-place, không trả về |
| `Concat` | **1.22** | `nil` nếu kết quả rỗng |
| `All`, `Backward`, `Values` | **1.23** | trả `iter.Seq`/`iter.Seq2` |
| `Collect`, `AppendSeq`, `Sorted`, `SortedFunc`, `SortedStableFunc` | **1.23** | nhận `iter.Seq` |
| `Chunk` | **1.23** | `iter.Seq[Slice]`, panic nếu `n < 1` |
| `Repeat` | **1.23** | không bao giờ `nil`; panic nếu overflow |

```go
s := []int{3, 1, 2, 2}

slices.Sort(s)                       // [1 2 2 3]
i, found := slices.BinarySearch(s, 2) // 1, true — pos để insert nếu !found
s = slices.Compact(s)                // [1 2 3]
s = slices.Insert(s, 1, 9)           // [1 9 2 3]
s = slices.Delete(s, 1, 2)           // [1 2 3]
dst := slices.Clone(s)               // backing độc lập

slices.SortFunc(people, func(a, b Person) int {
	return cmp.Or(cmp.Compare(a.Last, b.Last), cmp.Compare(a.First, b.First))
})
```

- `SortFunc` yêu cầu `cmp` là **strict weak ordering** (trả 0 cho phần tử không so sánh được); comparator sai luật có thể làm sort panic hoặc cho kết quả vô nghĩa.
- Cần giữ thứ tự tương đối của phần tử bằng nhau → `SortStableFunc`.
- `cmp.Or` (1.22) rất tiện để xếp theo nhiều khóa như trên.

### 10.2 Bảng tra `maps` theo version

| Hàm | Version | Ghi chú |
|---|---|---|
| `Clone` | 1.21 | shallow; `nil` → `nil` |
| `Copy(dst, src)` | 1.21 | ghi đè key trùng |
| `Equal`, `EqualFunc` | 1.21 | `Equal(nil, map[K]V{})` → `true` |
| `DeleteFunc` | 1.21 | xóa tại chỗ |
| `Keys`, `Values` | **1.23** | trả `iter.Seq[K]` / `iter.Seq[V]` |
| `All` | **1.23** | `iter.Seq2[K, V]` |
| `Collect` | **1.23** | `iter.Seq2[K,V]` → `map[K]V` |
| `Insert(m, seq)` | **1.23** | nạp `iter.Seq2` vào map có sẵn |

Đúng cho Go 1.26: `maps.Keys` và `maps.Values` trả về **iterator**, không phải slice. Bản trả slice chỉ tồn tại ở `golang.org/x/exp/maps` (không thuộc stdlib) — đừng lẫn hai API.

```go
m := map[string]int{"b": 2, "a": 1, "c": 3}

ks := slices.Collect(maps.Keys(m))    // []string, thứ tự bất kỳ
sorted := slices.Sorted(maps.Keys(m)) // []string đã sort: [a b c]
sum := 0
for v := range maps.Values(m) {       // range trực tiếp trên iterator
	sum += v
}

cp := maps.Clone(m)
maps.DeleteFunc(cp, func(k string, v int) bool { return v == 0 })
back := maps.Collect(maps.All(m))     // roundtrip: maps.Equal(m, back) == true
```

Đảo key/value, hoặc dựng map từ slice:

```go
inv := make(map[int]string, len(m))
for k, v := range m {
	inv[v] = k
}

byID := maps.Collect(func(yield func(int, User) bool) {
	for _, u := range users {
		if !yield(u.ID, u) {
			return
		}
	}
})
```

### 10.3 Bẫy: hàm ghi vào slice gốc

Nhiều hàm `slices` **sửa tại chỗ** rồi trả slice ngắn hơn. Slice cũ vẫn trỏ vào cùng backing nên nội dung của nó cũng đổi:

```go
s := []int{1, 1, 2, 2, 3}
out := slices.Compact(s)
// out = [1 2 3]
// s   = [1 2 3 0 0]  ← đã bị ghi, phần đuôi bị zero

d := []int{1, 2, 3, 4, 5}
_ = slices.Delete(d, 1, 3)
// d = [1 4 5 0 0]
```

- Việc zero phần đuôi là **có chủ ý**: tránh giữ pointer chết (§4.1). Nhưng nó nghĩa là dùng lại `s` sau `Compact`/`Delete` là sai.
- Quy tắc: coi các hàm này như `append` — luôn `s = slices.Compact(s)`; muốn giữ bản gốc thì `slices.Clone` trước.
- `Sort`, `Reverse`, `DeleteFunc` cũng sửa tại chỗ. `Clone`, `Concat`, `Repeat`, `Collect`, `Sorted` thì cấp phát mới.

---

## 11. Iterator trên collection (Go 1.23+)

`iter.Seq[V] = func(yield func(V) bool)` và `iter.Seq2[K, V] = func(yield func(K, V) bool)`. `range` chấp nhận đúng shape đó — xem [typesystem.md](typesystem.md) §13 và [statements.md](statements.md).

```go
for i, v := range slices.All(s)      { }   // iter.Seq2[int, E] — như range s
for i, v := range slices.Backward(s) { }   // duyệt ngược
for v := range slices.Values(s)      { }   // chỉ value
for k := range maps.Keys(m)          { }
for k, v := range maps.All(m)        { }
for chunk := range slices.Chunk(s, 100) { } // []E, mỗi lô ≤ 100
```

Vì sao hữu ích khi đã có `range s`: iterator **ghép được** và không cần vật chất hóa slice trung gian.

```go
// Không cấp phát slice keys trung gian
n := 0
for k := range maps.Keys(m) {
	if strings.HasPrefix(k, "user:") {
		n++
	}
}

// Xử lý theo lô, không tự tính index
for batch := range slices.Chunk(rows, 500) {
	if err := db.InsertMany(batch); err != nil {
		return err // break sớm: yield trả false, producer dừng
	}
}
```

- `slices.Chunk` clip mỗi lô (`cap == len`) nên `append` vào lô không đè lô sau.
- `break`/`return` trong `range` làm `yield` trả `false`; producer phải `return` ngay. Gọi `yield` sau khi nó trả `false` là **panic**.
- Thu về collection: `slices.Collect`, `slices.Sorted`, `slices.AppendSeq`, `maps.Collect`, `maps.Insert`.

**`iter.Pull`** đổi push-iterator thành pull — cần khi phải đọc từ hai nguồn xen kẽ (merge, zip, lookahead):

```go
next, stop := iter.Pull(slices.Values([]int{10, 20, 30}))
defer stop() // BẮT BUỘC nếu không đọc hết

v1, ok1 := next() // 10, true
v2, ok2 := next() // 20, true
```

- Sau khi hết chuỗi, `next` trả `(zero, false)` mãi mãi; gọi `next`/`stop` từ nhiều goroutine cùng lúc là lỗi.
- `iter.Pull2` cho `Seq2`.
- Không quên `stop()` → producer có thể bị treo giữ resource (`iter.Pull` chạy producer trên goroutine riêng).

Tự viết iterator cho type của mình (quy ước tên: `All`, `Backward`, `Values`):

```go
type Ring[T any] struct{ items []T; start int }

func (r *Ring[T]) All() iter.Seq2[int, T] {
	return func(yield func(int, T) bool) {
		for i := range r.items {
			j := (r.start + i) % len(r.items)
			if !yield(i, r.items[j]) {
				return
			}
		}
	}
}
```

---

## 12. `sync.Map` (tóm tắt)

`sync.Map` — map concurrent-safe trong `sync`, key/value kiểu `any`:

- Phù hợp: key ổn định, write-once / read-many, hoặc disjoint key giữa các goroutine.
- API: `Store`, `Load`, `LoadOrStore`, `LoadAndDelete` (1.15), `Delete`, `Range`, `Swap`/`CompareAndSwap`/`CompareAndDelete` (1.20), `Clear` (1.23).
- **Không** phải thay thế mặc định cho `map` + `Mutex` — mất type safety, mỗi value bị box vào `any`; benchmark trước.

```go
var sm sync.Map
sm.Store("a", 1)
if v, ok := sm.Load("a"); ok {
	fmt.Println(v.(int))
}
sm.Range(func(k, v any) bool { return true }) // false để dừng sớm
```

Muốn vừa type-safe vừa concurrent, bọc map thường bằng generic:

```go
type SyncMap[K comparable, V any] struct {
	mu sync.RWMutex
	m  map[K]V
}

func (s *SyncMap[K, V]) Load(k K) (V, bool) {
	s.mu.RLock()
	defer s.mu.RUnlock()
	v, ok := s.m[k]
	return v, ok
}
```

Map thường **không** an toàn khi concurrent read+write: runtime phát hiện và báo `fatal error: concurrent map writes` / `concurrent map read and map write` — đây là **fatal error**, `recover` không cứu được. Chi tiết concurrency: [concurrency.md](concurrency.md).

---

## 13. Pitfalls & best practices

1. Quên `s = append(s, ...)` → mất phần tử mới.
2. Subslice giữ sống cả backing lớn → memory leak logic; `slices.Clone` khi giữ lâu.
3. Append vào subslice đè dữ liệu “đuôi”. Dùng full slice `s[i:j:j]`.
4. `copy(dst, src)` với `dst` có `len == 0` copy 0 phần tử — `make([]T, len(src))`, không phải `make([]T, 0, len(src))`.
5. Dùng lại slice gốc sau `slices.Compact`/`Delete`/`Sort` — chúng sửa tại chỗ.
6. Map không an toàn cho concurrent read+write → mutex hoặc `sync.Map`.
7. Không dựa vào thứ tự `range` map, và cũng đừng coi nó là random source.
8. Sửa struct trong `map[K]V` → không compile; dùng read-modify-write hoặc `map[K]*V`.
9. Float làm map key: `NaN` sinh entry không xóa được.
10. Phân biệt nil/empty khi JSON; kiểm tra `len(s) > 0` chứ không `s != nil`.
11. Preallocate `make([]T, 0, n)` / `make(map[K]V, n)` khi biết n.
12. Array lớn trên stack có thể đắt — cân nhắc slice hoặc `*[N]T`.
13. `maps.Keys` trả iterator, không phải slice — bọc `slices.Collect`/`slices.Sorted` khi cần slice.

### Cheat sheet

| Thao tác | Cú pháp |
|----------|---------|
| Tạo slice | `make([]T, len, cap)` |
| Thêm | `s = append(s, x)` |
| Clone | `slices.Clone(s)` (1.21) |
| Copy vào chỗ có sẵn | `copy(dst, src)` — `len(dst)` quyết định |
| Giới hạn cap | `s[i:j:j]` / `slices.Clip(s)` |
| Xóa index i | `slices.Delete(s, i, i+1)` (1.21) |
| Sort | `slices.Sort(s)` / `slices.SortFunc(s, cmp)` |
| Sort nhiều khóa | `cmp.Or(cmp.Compare(...), ...)` (1.22) |
| Tìm trong slice đã sort | `slices.BinarySearch(s, x)` |
| Chia lô | `slices.Chunk(s, n)` (1.23) |
| Xóa map key | `delete(m, k)` |
| Có key? | `v, ok := m[k]` |
| Xóa hết | `clear(m)` / `clear(s)` (1.21) |
| Key đã sort | `slices.Sorted(maps.Keys(m))` (1.23) |
| Iterator → slice/map | `slices.Collect` / `maps.Collect` (1.23) |
| Pull iterator | `next, stop := iter.Pull(seq)` (1.23) |
| Nil slice JSON | `null` vs `[]` |
