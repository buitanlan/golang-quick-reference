# Generics trong Go

Generics (type parameters) từ **Go 1.18** cho phép viết hàm/kiểu tham số hóa kiểu mà vẫn type-safe lúc biên dịch. Dùng đúng chỗ giảm trùng lặp; lạm dụng làm API khó đọc.

Tài liệu nhắm tới **Go 1.26**. Type system / alias / comparable: [typesystem.md](typesystem.md). Method set: [methods-interfaces.md](methods-interfaces.md).

---

## Mục lục

- [Generics trong Go](#generics-trong-go)
  - [Mục lục](#mục-lục)
  - [1. Tổng quan](#1-tổng-quan)
  - [2. Type parameters](#2-type-parameters)
  - [3. Constraints](#3-constraints)
  - [4. `any` \& `comparable`](#4-any--comparable)
  - [5. Toán tử `~` \& underlying type](#5-toán-tử--underlying-type)
  - [6. Type sets](#6-type-sets)
  - [7. Generic functions](#7-generic-functions)
  - [8. Generic types \& type aliases](#8-generic-types--type-aliases)
  - [9. Method \& giới hạn](#9-method--giới-hạn)
  - [10. Type inference](#10-type-inference)
  - [11. Self-referential constraints (Go 1.26+)](#11-self-referential-constraints-go-126)
  - [12. Stdlib generic hữu ích](#12-stdlib-generic-hữu-ích)
  - [13. Ví dụ: `Min`, `Stack`, `Set`](#13-ví-dụ-min-stack-set)
  - [14. Khi nào KHÔNG dùng generics](#14-khi-nào-không-dùng-generics)
  - [15. Best practices](#15-best-practices)

---

## 1. Tổng quan

Trước generics: `interface{}` + type assert, hoặc code gen, hoặc copy-paste theo kiểu.

```go
// Cũ
func MinInt(a, b int) int { if a < b { return a }; return b }
func MinFloat64(a, b float64) float64 { /* ... */ }

// Mới
func Min[T cmp.Ordered](a, b T) T {
	if a < b {
		return a
	}
	return b
}
```

- Tham số kiểu trong `[...]` trước danh sách tham số giá trị.
- Constraint giới hạn type set cho phép toán tử / method.
- Instantiation: `Min[int](1, 2)` hoặc suy luận `Min(1, 2)`.
- Go **1.25** không thêm cú pháp ngôn ngữ mới; spec bỏ khái niệm “core type” (mục 6). Go **1.26** thêm constraint tự tham chiếu (mục 11).

---

## 2. Type parameters

```go
func Print[T any](v T) {
	fmt.Println(v)
}

type Pair[K any, V any] struct {
	Key   K
	Value V
}
```

- Tên thường `T`, `K`, `V`, `E`, `S ~[]E`…
- Nhiều tham số: `[T any, U cmp.Ordered]`.
- Type parameter là **kiểu** — dùng như tên kiểu trong thân.

---

## 3. Constraints

Constraint = interface (có thể liệt kê type set):

```go
type Number interface {
	int | int64 | float64
}

func Sum[T Number](xs []T) T {
	var total T
	for _, x := range xs {
		total += x
	}
	return total
}
```

- Interface “thường” (method) cũng là constraint.
- Có thể **giao** type set với method: kiểu phải nằm trong set **và** có đủ method.

Ví dụ biên dịch được — named slice `[]byte` có method `String`:

```go
type Bytes interface {
	~[]byte
	String() string
}

type B []byte

func (b B) String() string { return string(b) }

func AsString[T Bytes](t T) string { return t.String() }

var _ = AsString[B] // OK
// AsString[[]byte] — LỖI: []byte không có method String
```

Tương tự `~string` + `String() string`: built-in `string` **không** thỏa (thiếu method); chỉ named string type có `String()` mới thỏa.

```go
type StringerIface interface {
	~string
	String() string
}
// F[string] → "string does not satisfy StringerIface (missing method String)"
```

Stdlib hữu ích:

```go
import "cmp" // Ordered, Less, Compare, Or (Or từ Go 1.22+)
```

Package `constraints` (x/exp cũ) hầu như không cần nữa: `cmp.Ordered` (1.21+) đủ cho `<` `>` trên số/string.

```go
import "cmp"

func Max[T cmp.Ordered](a, b T) T {
	if a > b {
		return a
	}
	return b
}
```

---

## 4. `any` & `comparable`

| Constraint | Nghĩa |
|------------|--------|
| `any` | alias `interface{}` — mọi kiểu |
| `comparable` | dùng được `==` `!=` (key map,…) |

```go
func Keys[K comparable, V any](m map[K]V) []K {
	keys := make([]K, 0, len(m))
	for k := range m {
		keys = append(keys, k)
	}
	return keys
}
```

- `comparable` **không** gồm slice/map/func.
- Struct comparable nếu mọi field comparable.
- Từ **Go 1.20**, interface thỏa `comparable` (có thể panic runtime nếu dynamic non-comparable) — [methods-interfaces.md](methods-interfaces.md) §12.2, [typesystem.md](typesystem.md) §9.1.

`cmp.Or` (Go 1.22+) — chọn giá trị đầu khác zero (cần `comparable`):

```go
addr := cmp.Or(cfg.Addr, ":8080")
name := cmp.Or(userName, "guest")
```

`any` khi chỉ cần lưu/truyền, không gọi method/toán tử đặc thù.

---

## 5. Toán tử `~` & underlying type

`~T` = mọi kiểu có **underlying type** là `T`:

```go
type MyInt int
type MyString string

type Intish interface {
	~int
}

func Double[T Intish](v T) T { return v * 2 }

var x MyInt = 3
fmt.Println(Double(x)) // OK — underlying int
```

Không có `~`:

```go
type OnlyInt interface{ int }
// Double với OnlyInt không nhận MyInt
```

Slice constraint phổ biến:

```go
func Last[S ~[]E, E any](s S) E {
	return s[len(s)-1]
}
```

- `S ~[]E` giữ được kiểu named slice khi trả về cùng `S` trong API phức tạp hơn.

---

## 6. Type sets

Interface constraint định nghĩa **tập kiểu** (type set) được phép:

```go
type OrderedNumeric interface {
	~int | ~int64 | ~float64
}
```

- Union `|`.
- Intersection = nhiều embed interface / liệt kê method + type element cùng interface.
- Type set của interface chỉ có method = mọi kiểu implement method đó.
- Empty interface / `any` → mọi kiểu.

### 6.1 Toán tử trên type set (không còn “core type”)

Kiểm tra tại compile: toán tử phải hợp lệ với **mọi** kiểu trong set.

Trước đây spec nói tới “core type” của type parameter để quyết định phép toán. Từ **Go 1.25**, khái niệm đó bị **bỏ** khỏi spec — luật được diễn đạt trực tiếp trên type set (cùng ý thực dụng: phép toán phải được mọi thành viên hỗ trợ).

```go
type Summable interface {
	int | string
}

func Add[T Summable](a, b T) T {
	return a + b // OK: cả int và string đều có +
}
```

```go
type NotSummable interface {
	int | bool
}

func AddBad[T NotSummable](a, b T) T {
	return a + b
	// LỖI: invalid operation: operator + not defined on a
	// (variable of type T constrained by NotSummable)
}
```

Quy tắc: union càng rộng, phép toán càng dễ bị từ chối — thu hẹp set hoặc tách hàm theo nhóm kiểu.

---

## 7. Generic functions

```go
func Map[T any, U any](in []T, f func(T) U) []U {
	out := make([]U, len(in))
	for i, v := range in {
		out[i] = f(v)
	}
	return out
}

nums := Map([]int{1, 2, 3}, func(x int) string {
	return fmt.Sprintf("#%d", x)
})
```

Filter:

```go
func Filter[T any](in []T, keep func(T) bool) []T {
	var out []T
	for _, v := range in {
		if keep(v) {
			out = append(out, v)
		}
	}
	return out
}
```

Nhiều thao tác đã có sẵn trong `slices` / `maps` (mục 12) — đừng viết lại nếu stdlib đủ.

---

## 8. Generic types & type aliases

```go
type Stack[T any] struct {
	items []T
}

func (s *Stack[T]) Push(v T) {
	s.items = append(s.items, v)
}

func (s *Stack[T]) Pop() (T, bool) {
	var zero T
	if len(s.items) == 0 {
		return zero, false
	}
	i := len(s.items) - 1
	v := s.items[i]
	s.items = s.items[:i]
	return v, true
}
```

Instantiation:

```go
var st Stack[string]
st.Push("a")
```

Pointer generic:

```go
type Ptr[T any] struct{ v *T }
```

### 8.1 Generic type alias (Go 1.24+)

Từ Go 1.24, **type alias** nhận được type parameter (cần language version đủ mới trong `go.mod`):

```go
type Set[T comparable] = map[T]struct{}
type List[T any]       = []T

s := Set[string]{"a": {}} // identical với map[string]struct{}
var l List[int] = []int{1}
```

- Alias → identical với kiểu bên phải → interop tự nhiên với API cũ.
- Muốn gắn method / đóng gói invariant → dùng **type definition** (`type Set[T comparable] map[T]struct{}`), không phải alias.
- Chi tiết: [typesystem.md](typesystem.md) §4.1.

---

## 9. Method & giới hạn

**Được:** method trên generic type dùng type params của type:

```go
func (s *Stack[T]) Len() int { return len(s.items) }
```

**Không được:** thêm type parameter mới trên method:

```go
// Illegal — lỗi: method must have no type parameters
// func (s *Stack[T]) Convert[U any]() Stack[U]
```

Workaround: hàm package-level:

```go
func Convert[T any, U any](s Stack[T], f func(T) U) Stack[U] {
	var out Stack[U]
	for _, v := range s.items {
		out.Push(f(v))
	}
	return out
}
```

Giới hạn khác:

- Không generic trên method của non-generic type.
- Specialization kiểu C++ không có — một implementation cho mọi T (compiler có thể instantiate/shape riêng, nhưng không viết overload theo T).
- Reflection vẫn động; generics là compile-time.
- Type switch trực tiếp trên type parameter không được — cast qua `any` — [typesystem.md](typesystem.md) §8.

---

## 10. Type inference

```go
Min(1, 2)           // T = int
Map(xs, fn)         // suy từ xs và fn
Stack[int]{}        // type cần chỉ rõ khi không có đối số suy ra
```

- Inference từ argument; return type ít khi đủ một mình.
- Chỉ rõ khi mơ hồ: `Min[int64](a, b)`.
- Constraint quá hẹp/rộng ảnh hưởng inference.

---

## 11. Self-referential constraints (Go 1.26+)

Constraint có thể tham chiếu chính type parameter đang khai báo — hữu ích cho API “trả về cùng kiểu nhận”:

```go
type Adder[A Adder[A]] interface {
	Add(A) A
}

type Nat int

func (n Nat) Add(other Nat) Nat { return n + other }

func Sum[A Adder[A]](a, b A) A {
	return a.Add(b)
}

var _ = Sum[Nat] // OK
```

- Pattern quen thuộc từ interface “fluent” / CRTP-like: `Add` nhận và trả **cùng** `A`, không phải interface rộng hơn.
- Trước 1.26, thường phải viết constraint kém chặt hoặc nhường cho interface thường + type assert.

---

## 12. Stdlib generic hữu ích

| API | Từ | Việc |
|-----|-----|------|
| `cmp.Ordered`, `cmp.Less`, `cmp.Compare` | 1.21 | so sánh ordered |
| `cmp.Or` | 1.22 | first non-zero |
| `slices.*` (`Equal`, `Sort`, `Clone`, …) | 1.21 | thao tác slice |
| `maps.*` (`Equal`, `Clone`, `Copy`, …) | 1.21 | thao tác map |
| `sync.OnceValue` / `OnceValues` | 1.21 | lazy init generic |
| `unique.Make` | 1.23 | intern giá trị comparable |
| `weak.Pointer` | 1.24 | weak ref |
| `runtime.AddCleanup` | 1.24 | cleanup gắn object (thay một số dùng finalizer) |
| `errors.AsType[E]` | 1.26 | lấy typed error khỏi cây wrap |
| `reflect.TypeFor[T]` | 1.22 | `Type` không cần value |

```go
import (
	"cmp"
	"errors"
	"slices"
	"sync"
	"unique"
)

_ = slices.Equal([]int{1, 2}, []int{1, 2})
_ = cmp.Or("", "default")

once := sync.OnceValue(func() int { return expensiveInit() })
_ = once()

h := unique.Make("canonical")
_ = h.Value()

if pe, ok := errors.AsType[*fs.PathError](err); ok {
	_ = pe.Path
}
```

`weak.Pointer[T]` (1.24): không giữ object sống; `Value()` có thể về `nil` sau GC — cache / canonical map. Kết hợp `unique` khi cần identity ổn định cho giá trị comparable.

---

## 13. Ví dụ: `Min`, `Stack`, `Set`

```go
package util

import "cmp"

func Min[T cmp.Ordered](a, b T) T {
	if a < b {
		return a
	}
	return b
}

func MinAll[T cmp.Ordered](first T, rest ...T) T {
	m := first
	for _, v := range rest {
		if v < m {
			m = v
		}
	}
	return m
}
```

```go
type Set[T comparable] map[T]struct{}

func NewSet[T comparable](elems ...T) Set[T] {
	s := make(Set[T], len(elems))
	for _, e := range elems {
		s[e] = struct{}{}
	}
	return s
}

func (s Set[T]) Has(v T) bool {
	_, ok := s[v]
	return ok
}

func (s Set[T]) Add(v T) { s[v] = struct{}{} }
```

Hoặc alias (1.24+) nếu không cần method: `type Set[T comparable] = map[T]struct{}`.

---

## 14. Khi nào KHÔNG dùng generics

1. **Chỉ một kiểu cụ thể** — hàm thường dễ đọc hơn.
2. **Hành vi khác nhau nhiều theo T** — interface + polymorphism rõ hơn “một thân generic”.
3. **API public sớm** — generic khó evolve hơn; đợi use-case ổn định.
4. **Chỉ để tránh 3 dòng lặp** — abstraction đắt hơn lợi.
5. **Cần specialize hiệu năng cực đoan theo T** — generics không cho specialization tường minh; có thể dùng `int` path riêng.
6. **Đã có interface nhỏ đủ dùng** (`io.Reader`) — đừng generic hóa lại.
7. **Stdlib đã có** (`slices.Contains`, `maps.Clone`, …) — gọi sẵn thay viết generic nội bộ trùng.

> Quy tắc ngón tay cái: viết concrete trước; generic hóa khi thấy **3+ bản copy** giống nhau chỉ khác kiểu.

---

## 15. Best practices

1. Đặt tên type param ngắn, constraint rõ (`S ~[]E`).
2. Dùng `cmp.Ordered`, `comparable`, `cmp.Or` từ std thay constraint tự chế khi đủ.
3. Trả zero value bằng `var zero T` (hoặc `*new(T)` / zero composite tùy ngữ cảnh).
4. Document complexity nếu container generic.
5. Test instantiate vài kiểu (int, string, custom named).
6. Đừng nhồi `any` constraint rồi type assert bên trong — đó không phải generics đúng nghĩa.
7. Alias generic (1.24) cho identical shorthand; definition khi cần method.
8. Self-referential constraint (1.26) khi API yêu cầu cùng kiểu nhận/trả.

### Cheat sheet

| Chủ đề | Gợi nhớ |
|--------|---------|
| Cú pháp | `func F[T C](v T)` |
| Mọi kiểu | `any` |
| `==` | `comparable` (1.20: interface thỏa được) |
| Ordered / default | `cmp.Ordered`, `cmp.Or` |
| Underlying | `~int`, `~[]E` |
| Union + toán tử | mọi thành viên phải hỗ trợ (`int\|string` OK với `+`; `int\|bool` không) |
| Type set ∩ method | named type trong set **và** có method |
| Alias generic | Go 1.24+ `type A[T any] = ...` |
| Self-ref constraint | Go 1.26+ `type C[T C[T]] interface{...}` |
| Method thêm `[U]` | không được |
| Core type | bỏ khỏi spec từ 1.25 — nghĩ bằng type set |
| Inference | từ arguments |
| Stdlib | `slices`, `maps`, `unique`, `weak`, `errors.AsType` |
| Tránh | generic sớm / một-shot / trùng stdlib |
