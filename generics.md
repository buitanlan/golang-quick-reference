# Generics trong Go

Generics (type parameters) từ **Go 1.18** cho phép viết hàm/kiểu tham số hóa kiểu mà vẫn type-safe lúc biên dịch. Dùng đúng chỗ giảm trùng lặp; lạm dụng làm API khó đọc.

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
  - [8. Generic types](#8-generic-types)
  - [9. Method \& giới hạn](#9-method--giới-hạn)
  - [10. Type inference](#10-type-inference)
  - [11. Ví dụ: `Min`, `Stack`](#11-ví-dụ-min-stack)
  - [12. Khi nào KHÔNG dùng generics](#12-khi-nào-không-dùng-generics)
  - [13. Best practices](#13-best-practices)

---

## 1. Tổng quan

Trước generics: `interface{}` + type assert, hoặc code gen, hoặc copy-paste theo kiểu.

```go
// Cũ
func MinInt(a, b int) int { if a < b { return a }; return b }
func MinFloat64(a, b float64) float64 { ... }

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
- Nhiều tham số: `[T any, U constraints.Integer]`.  
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
- Kết hợp method + type set:

```go
type StringerIface interface {
	~string
	String() string // ví dụ minh họa — string thường không có String(); ý tưởng kết hợp
}
```

Stdlib hữu ích:

```go
import "cmp"           // Ordered (Go 1.21+)
import "constraints"   // trước 1.21; một phần chuyển sang cmp
```

Go 1.21+: `cmp.Ordered` cho `<` `>` trên số/string.

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
- Interface có thể comparable runtime — cẩn thận panic nếu dynamic non-comparable.

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
// DoubleOnly[OnlyInt] không nhận MyInt
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
- Intersection = nhiều embed interface.  
- Type set của interface method thường = mọi kiểu implement method.  
- Empty interface / `any` → mọi kiểu.

Kiểm tra tại compile: toán tử phải hợp lệ với **mọi** kiểu trong set.

```go
// Lỗi: string không hỗ trợ nếu chỉ muốn + số
type Bad interface {
	int | string
}
func Add[T Bad](a, b T) T { return a + b } // OK thực ra với int|string vì cả hai có +
// Nhưng int | bool → + lỗi
```

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

---

## 8. Generic types

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

---

## 9. Method & giới hạn

**Được:** method trên generic type dùng type params của type:

```go
func (s *Stack[T]) Len() int { return len(s.items) }
```

**Không được:** thêm type parameter mới trên method:

```go
// Illegal:
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
- Specialization kiểu C++ không có — một implementation cho mọi T.  
- Reflection vẫn động; generics là compile-time.

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

## 11. Ví dụ: `Min`, `Stack`

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

---

## 12. Khi nào KHÔNG dùng generics

1. **Chỉ một kiểu cụ thể** — hàm thường dễ đọc hơn.  
2. **Hành vi khác nhau nhiều theo T** — interface + polymorphism rõ hơn “một thân generic”.  
3. **API public sớm** — generic khó evolve hơn; đợi use-case ổn định.  
4. **Chỉ để tránh 3 dòng lặp** — abstraction đắt hơn lợi.  
5. **Cần specialize hiệu năng cực đoan theo T** — generics không cho specialization; có thể dùng `int` path riêng.  
6. **Đã có interface nhỏ đủ dùng** (`io.Reader`) — đừng generic hóa lại.

> Quy tắc ngón tay cái: viết concrete trước; generic hóa khi thấy **3+ bản copy** giống nhau chỉ khác kiểu.

---

## 13. Best practices

1. Đặt tên type param ngắn, constraint rõ (`S ~[]E`).  
2. Dùng `cmp.Ordered`, `comparable` từ std thay constraint tự chế khi đủ.  
3. Trả zero value bằng `var zero T`.  
4. Document complexity nếu container generic.  
5. Test instantiate vài kiểu (int, string, custom).  
6. Đừng nhồi `any` constraint rồi type assert bên trong — đó không phải generics đúng nghĩa.

### Cheat sheet

| Chủ đề | Gợi nhớ |
|--------|---------|
| Cú pháp | `func F[T C](v T)` |
| Mọi kiểu | `any` |
| `==` | `comparable` |
| Underlying | `~int`, `~[]E` |
| Union | `int \| string` |
| Method thêm `[U]` | không được |
| Inference | từ arguments |
| Tránh | generic sớm / một-shot |
