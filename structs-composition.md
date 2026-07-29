# Struct & Composition trong Go

Struct là kiểu dữ liệu tổng hợp (composite type) chính của Go — nhóm các field thành một đơn vị. Go **không** có class/inheritance; thay vào đó dùng **composition** (embedding) và interface để tái sử dụng hành vi.

---

## Mục lục

- [Struct \& Composition trong Go](#struct--composition-trong-go)
  - [Mục lục](#mục-lục)
  - [1. Tổng quan](#1-tổng-quan)
  - [2. Khai báo struct](#2-khai-báo-struct)
  - [3. Field xuất khẩu (exported) vs không xuất khẩu](#3-field-xuất-khẩu-exported-vs-không-xuất-khẩu)
  - [4. Struct literal \& khởi tạo](#4-struct-literal--khởi-tạo)
  - [5. Anonymous field \& embedding](#5-anonymous-field--embedding)
  - [6. Composition vs inheritance](#6-composition-vs-inheritance)
  - [7. Promotion: field \& method](#7-promotion-field--method)
  - [8. Tag (json, xml, db…)](#8-tag-json-xml-db)
  - [9. So sánh struct](#9-so-sánh-struct)
  - [10. Empty struct `struct{}`](#10-empty-struct-struct)
  - [11. Constructor convention: `NewXxx`](#11-constructor-convention-newxxx)
  - [12. Pointer receiver trên struct](#12-pointer-receiver-trên-struct)
  - [13. Best practices](#13-best-practices)

---

## 1. Tổng quan

- Struct là **value type**: gán/truyền hàm **copy** toàn bộ field (trừ khi dùng pointer).  
- Không có constructor ngôn ngữ; convention là hàm `NewXxx` / `New`.  
- Embedding cho phép “nâng” (promote) field/method của kiểu nhúng — gần giống kế thừa nhưng **không** phải is-a.  
- Tag gắn metadata cho reflection (`encoding/json`, ORM…).

> **Lưu ý**: Zero value của struct = zero value của từng field. Struct luôn “có giá trị”; không có `null` struct — chỉ có `nil` khi dùng *pointer* tới struct.

---

## 2. Khai báo struct

```go
type Point struct {
	X int
	Y int
}

type Person struct {
	Name string
	Age  int
}
```

- Thứ tự field trong memory theo khai báo (có thể có padding alignment).  
- Có thể khai báo **anonymous struct** (một lần dùng):

```go
cfg := struct {
	Host string
	Port int
}{
	Host: "localhost",
	Port: 8080,
}
```

Struct lồng nhau:

```go
type Address struct {
	City, Country string
}

type Employee struct {
	Name    string
	Address Address // named field
}
```

---

## 3. Field xuất khẩu (exported) vs không xuất khẩu

- Tên bắt đầu **chữ hoa** → exported (nhìn thấy ngoài package).  
- Chữ thường → unexported (chỉ trong package).

```go
package user

type User struct {
	ID       int    // exported
	Name     string // exported
	password string // unexported — encapsulation trong package
}
```

- `encoding/json` **chỉ** marshal/unmarshal field exported (trừ khi dùng custom `MarshalJSON`).  
- Cùng package vẫn truy cập được field unexported.

---

## 4. Struct literal & khởi tạo

```go
p1 := Point{1, 2}           // positional — dễ gãy khi đổi thứ tự field
p2 := Point{X: 1, Y: 2}     // keyed — khuyến nghị
p3 := Point{X: 1}           // Y = 0 (zero value)
var p4 Point                // X=0, Y=0

ptr := &Point{X: 10, Y: 20} // *Point
ptr2 := new(Point)          // *Point, zero value
```

- Literal keyed **an toàn** khi thêm/bớt field.  
- Positional chỉ nên dùng trong package nhỏ, test nội bộ.  
- Có thể lấy địa chỉ của literal: `&T{...}` (Go cấp phát trên heap nếu cần).

Cập nhật field:

```go
p := Point{X: 1, Y: 2}
p.X = 5

ptr := &p
ptr.Y = 9 // Go tự deref: (*ptr).Y = 9
```

---

## 5. Anonymous field & embedding

**Anonymous field** = field không đặt tên, chỉ khai báo kiểu:

```go
type Engine struct {
	Power int
}

func (e Engine) Start() { fmt.Println("engine start", e.Power) }

type Car struct {
	Engine // embedding — anonymous field
	Brand  string
}
```

- Tên field mặc định = tên kiểu (`Engine`).  
- Có thể embed pointer: `*Engine` — hữu ích khi optional / tránh copy lớn.

```go
type Car2 struct {
	*Engine
	Brand string
}
```

Nhiều cấp embedding:

```go
type A struct{ V int }
type B struct{ A }
type C struct{ B }

c := C{}
c.V = 42 // promote qua B → A
```

---

## 6. Composition vs inheritance

| Inheritance (OOP cổ điển) | Composition / embedding (Go) |
|---------------------------|------------------------------|
| is-a | has-a / “contains” |
| Override đa hình qua virtual | Không có virtual; dùng interface |
| Constructor chaining | `NewXxx` tự gọi `NewYyy` |
| Protected members | Unexported + cùng package |

```go
// Không: class Truck extends Vehicle
// Có: Truck chứa/embed Vehicle + implement interface Driver

type Vehicle struct {
	Wheels int
}

func (v Vehicle) Move() { fmt.Println("moving") }

type Truck struct {
	Vehicle
	Capacity int
}

func (t Truck) Move() { // method cùng tên — *che* method được promote, không phải override đa hình
	fmt.Println("truck moving", t.Capacity)
}
```

- Gọi qua kiểu concrete: `Truck.Move` dùng method của `Truck`.  
- Nếu gán vào interface chỉ có `Move`, hành vi phụ thuộc **kiểu động** của value — không có “virtual table từ base class”.

---

## 7. Promotion: field & method

Field/method của kiểu được embed được **promote** lên kiểu ngoài:

```go
c := Car{Engine: Engine{Power: 100}, Brand: "Vin"}
fmt.Println(c.Power) // == c.Engine.Power
c.Start()            // == c.Engine.Start()
```

**Xung đột tên**:

```go
type Inner struct{ Name string }
type Outer struct {
	Inner
	Name string // che Inner.Name khi truy cập Outer.Name
}

o := Outer{Inner: Inner{Name: "in"}, Name: "out"}
fmt.Println(o.Name)       // "out"
fmt.Println(o.Inner.Name) // "in" — truy cập tường minh
```

- Hai anonymous field cùng promote một tên → **ambiguous**; phải chỉ rõ đường dẫn.  
- Method của outer cùng tên với method promote → outer thắng khi gọi trên outer.

Selector rules (tóm tắt):

1. Field/method khai báo trực tiếp trên `T`.  
2. Field/method promote từ embed depth nhỏ hơn.  
3. Ambiguous nếu nhiều ứng viên cùng depth.

---

## 8. Tag (json, xml, db…)

Tag là chuỗi literal sau field, parse bởi `reflect`:

```go
type UserDTO struct {
	ID        int    `json:"id"`
	Name      string `json:"name"`
	Password  string `json:"-"`              // bỏ qua
	Email     string `json:"email,omitempty"` // bỏ nếu zero value
	CreatedAt string `json:"created_at"`
}
```

```go
import (
	"encoding/json"
	"fmt"
)

u := UserDTO{ID: 1, Name: "An", Password: "secret"}
b, _ := json.Marshal(u)
fmt.Println(string(b)) // {"id":1,"name":"An","created_at":""}
```

Tag phổ biến:

| Tag | Ý nghĩa |
|-----|---------|
| `json:"name"` | tên field JSON |
| `json:"-"` | bỏ qua |
| `json:",omitempty"` | bỏ zero value |
| `json:",string"` | encode số thành string |
| `db:"col"` / `gorm:"..."` | ORM |

- Nhiều tag: `` `json:"id" db:"user_id"` ``  
- Tag sai chính tả → thư viện bỏ qua im lặng — kiểm tra bằng test marshal.

---

## 9. So sánh struct

Hai struct **cùng kiểu** so sánh được bằng `==` nếu **mọi field comparable**:

```go
type P struct{ X, Y int }
fmt.Println(P{1, 2} == P{1, 2}) // true
```

**Không comparable** nếu chứa: slice, map, function:

```go
type Bad struct {
	Data []byte
}
// Bad{} == Bad{} → compile error
```

- Dùng `reflect.DeepEqual` (chậm) hoặc so từng field / thư viện.  
- Array comparable nếu element comparable.  
- Interface field: comparable nếu dynamic value comparable.

```go
type Box struct {
	V any
}
// Box{1} == Box{1} OK; Box{[]int{1}} == ... panic lúc runtime nếu so sánh non-comparable
```

---

## 10. Empty struct `struct{}`

`struct{}` chiếm **0 byte**; dùng làm:

- Signal trên channel: `chan struct{}`  
- Set giả lập: `map[T]struct{}`  
- Marker type không cần dữ liệu

```go
done := make(chan struct{})
go func() {
	// ...
	close(done) // hoặc done <- struct{}{}
}()
<-done

set := map[string]struct{}{}
set["a"] = struct{}{}
if _, ok := set["a"]; ok {
	fmt.Println("có a")
}
```

> Nhiều `struct{}{}` literal có thể dùng chung địa chỉ (implementation detail) — đừng dựa vào pointer identity cho empty struct trừ khi đọc rõ spec/runtime.

---

## 11. Constructor convention: `NewXxx`

Go không có `new` kiểu OOP; convention:

```go
type Config struct {
	Host string
	Port int
}

// NewConfig trả về giá trị hoặc pointer + error
func NewConfig(host string, port int) (*Config, error) {
	if host == "" {
		return nil, fmt.Errorf("host rỗng")
	}
	if port <= 0 {
		return nil, fmt.Errorf("port không hợp lệ: %d", port)
	}
	return &Config{Host: host, Port: port}, nil
}
```

Quy ước phổ biến:

- `NewT(...) (*T, error)` — khởi tạo có validation.  
- `MustNewT(...) *T` — panic nếu lỗi (init, test).  
- Trả **pointer** khi struct lớn, có mutex, hoặc cần identity chia sẻ.  
- Trả **value** khi nhỏ, bất biến, copy rẻ.

Functional options (pattern nâng cao):

```go
type Option func(*Server)

func WithAddr(addr string) Option {
	return func(s *Server) { s.Addr = addr }
}

func NewServer(opts ...Option) *Server {
	s := &Server{Addr: ":8080"} // default
	for _, opt := range opts {
		opt(s)
	}
	return s
}
```

---

## 12. Pointer receiver trên struct

```go
type Counter struct{ n int }

func (c *Counter) Inc() { c.n++ }
func (c Counter) Value() int { return c.n }
```

- Sửa state → pointer receiver.  
- Consistency: nếu một method dùng pointer, thường cả API nên thống nhất.  
- Embedding `*T` vs `T` ảnh hưởng nil-safety và copy.

Chi tiết receiver xem thêm [pointers.md](pointers.md).

---

## 13. Best practices

1. Dùng keyed struct literal trong API công khai.  
2. Ẩn field bằng unexported; expose qua method nếu cần invariant.  
3. Ưu tiên composition + interface hơn “cây kế thừa”.  
4. Tag `json` rõ ràng; viết test round-trip.  
5. `NewXxx` validate và trả `error` — đừng để zero value “nửa hợp lệ” nếu nguy hiểm.  
6. Tránh embed kiểu chỉ để “tiết kiệm gõ”; embed khi quan hệ has-a thật sự và muốn promotion.  
7. Không so sánh struct chứa slice/map bằng `==`.  
8. `struct{}` cho signal/set — đừng nhồi dữ liệu vào empty struct.

---

### Cheat sheet

| Chủ đề | Gợi nhớ |
|--------|---------|
| Zero value | Luôn hợp lệ ở mức field |
| Exported | Chữ hoa |
| Embed | Promotion field/method |
| Tag | Reflect metadata |
| `==` | Mọi field phải comparable |
| `struct{}` | 0 byte, signal/set |
| `NewXxx` | Constructor convention |
