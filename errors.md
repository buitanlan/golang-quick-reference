# Error handling trong Go

Go xử lý lỗi bằng giá trị (`error`), không phải exception. Caller **kiểm tra** `err != nil` tường minh. Panic dành cho lỗi lập trình / tình huống không thể tiếp tục — không thay try/catch cho luồng nghiệp vụ.

---

## Mục lục

- [Error handling trong Go](#error-handling-trong-go)
  - [Mục lục](#mục-lục)
  - [1. Triết lý](#1-triết-lý)
  - [2. Interface `error`](#2-interface-error)
  - [3. `errors.New` \& `fmt.Errorf`](#3-errorsnew--fmterrorf)
  - [4. Wrapping với `%w` (Go 1.13+)](#4-wrapping-với-w-go-113)
  - [5. `errors.Is` / `errors.As` / `Unwrap`](#5-errorsis--errorsas--unwrap)
  - [6. Sentinel errors](#6-sentinel-errors)
  - [7. Custom error types](#7-custom-error-types)
  - [8. `errors.Join` (Go 1.20+)](#8-errorsjoin-go-120)
  - [9. Panic \& recover](#9-panic--recover)
  - [10. `defer` + recover patterns](#10-defer--recover-patterns)
  - [11. Khi nào KHÔNG dùng panic](#11-khi-nào-không-dùng-panic)
  - [12. Best practices \& checklist](#12-best-practices--checklist)

---

## 1. Triết lý

- Lỗi là **giá trị** — có thể trả về, bọc, log, so sánh.  
- Multiple return: `(T, error)` — `error` thường ở cuối.  
- Không có stack unwind kiểu exception cho lỗi thường.  
- `if err != nil { return ..., err }` lặp lại là **cố ý** (rõ ràng tại call site).

```go
f, err := os.Open(path)
if err != nil {
	return fmt.Errorf("open %s: %w", path, err)
}
defer f.Close()
```

---

## 2. Interface `error`

```go
type error interface {
	Error() string
}
```

- Mọi kiểu có method `Error() string` đều là `error`.  
- `nil` error = thành công.  
- So sánh interface `err == nil` đúng khi không có giá trị động; **typed nil** có thể ≠ nil interface (tránh return `(*MyErr)(nil)` như error).

```go
func bad() error {
	var e *MyError = nil
	return e // trả về error interface ≠ nil!
}
func good() error {
	var e *MyError = nil
	if e == nil {
		return nil
	}
	return e
}
```

---

## 3. `errors.New` & `fmt.Errorf`

```go
import (
	"errors"
	"fmt"
)

err1 := errors.New("không tìm thấy")
err2 := fmt.Errorf("user %d không tồn tại", id)
```

- `errors.New` — chuỗi cố định / sentinel đơn giản.  
- `fmt.Errorf` — format động; dùng `%w` để wrap (mục 4).  
- `%v` / `%s` nhúng message **không** tạo chain unwrap.

---

## 4. Wrapping với `%w` (Go 1.13+)

```go
if err != nil {
	return fmt.Errorf("load config: %w", err)
}
```

- `%w` gắn lỗi gốc để `errors.Is` / `As` / `Unwrap` đi xuyên.  
- Chỉ **một** `%w` mỗi `fmt.Errorf` (trước Join; với Join xem mục 8).  
- Thêm ngữ cảnh (operation, id) ở mỗi tầng — đừng chỉ `return err` nếu mất context hữu ích.

```go
func ReadUser(id int) (User, error) {
	u, err := db.Find(id)
	if err != nil {
		return User{}, fmt.Errorf("ReadUser id=%d: %w", id, err)
	}
	return u, nil
}
```

`errors.Unwrap`:

```go
inner := errors.Unwrap(err) // lỗi được wrap, hoặc nil
```

---

## 5. `errors.Is` / `errors.As` / `Unwrap`

### `errors.Is(err, target)`

Đi theo chain; true nếu gặp lỗi **bằng** target (qua `==` hoặc method `Is`).

```go
if errors.Is(err, fs.ErrNotExist) {
	// file không tồn tại
}
```

### `errors.As(err, &target)`

Tìm lỗi trong chain gán được vào kiểu `target` trỏ tới:

```go
var pathErr *fs.PathError
if errors.As(err, &pathErr) {
	fmt.Println(pathErr.Path, pathErr.Op)
}
```

### Không dùng `==` / type assert mù quáng trên wrapped error

```go
// Yếu: mất khi đã wrap
if err == io.EOF { }

// Đúng:
if errors.Is(err, io.EOF) { }
```

---

## 6. Sentinel errors

Biến error package-level dùng làm “mã lỗi” ổn định:

```go
package user

import "errors"

var (
	ErrNotFound     = errors.New("user: not found")
	ErrInvalidEmail = errors.New("user: invalid email")
)

func Find(id int) (User, error) {
	// ...
	return User{}, ErrNotFound
}
```

Caller:

```go
u, err := user.Find(1)
if errors.Is(err, user.ErrNotFound) {
	// 404
}
```

- Xuất khẩu `ErrXxx`.  
- Message ổn định cho người; programmatic check dùng `Is`, không parse string.  
- Có thể implement `Is(error) bool` trên custom type để so sánh linh hoạt.

---

## 7. Custom error types

Khi cần field thêm (code HTTP, temporary, retry…):

```go
type ValidationError struct {
	Field   string
	Message string
}

func (e *ValidationError) Error() string {
	return fmt.Sprintf("%s: %s", e.Field, e.Message)
}

type QueryError struct {
	Query string
	Err   error
}

func (e *QueryError) Error() string {
	return fmt.Sprintf("query %q: %v", e.Query, e.Err)
}

func (e *QueryError) Unwrap() error { return e.Err }
```

Interface hữu ích (stdlib):

```go
// temporary / timeout — pattern cổ (net.Error)
type temporary interface {
	Temporary() bool
}
```

Hoặc dùng sentinel + wrap thay vì quá nhiều type.

`errors.As` với custom:

```go
var ve *ValidationError
if errors.As(err, &ve) {
	fmt.Println(ve.Field)
}
```

---

## 8. `errors.Join` (Go 1.20+)

Gộp nhiều lỗi thành một (ví dụ đóng nhiều resource, multi-validate):

```go
var errs []error
if e1 != nil {
	errs = append(errs, e1)
}
if e2 != nil {
	errs = append(errs, e2)
}
return errors.Join(errs...)
```

- `errors.Join` bỏ qua `nil`.  
- Trả `nil` nếu mọi input nil.  
- `errors.Is` / `As` đi qua **tất cả** lỗi con.  
- `Unwrap() []error` (multi-unwrap) — `Is`/`As` hỗ trợ.

```go
err := errors.Join(ErrA, ErrB)
fmt.Println(errors.Is(err, ErrA)) // true
fmt.Println(errors.Is(err, ErrB)) // true
```

Kết hợp với wrap:

```go
return fmt.Errorf("shutdown: %w", errors.Join(errClose, errFlush))
```

---

## 9. Panic & recover

**panic**: dừng stack bình thường, chạy deferred, lan lên caller cho tới recover hoặc crash process.

```go
panic("bug: invariant broken")
panic(fmt.Errorf("unexpected: %w", err))
```

**recover**: chỉ hữu hiệu bên trong **deferred** function:

```go
func safe() {
	defer func() {
		if r := recover(); r != nil {
			fmt.Println("recovered:", r)
		}
	}()
	panic("boom")
}
```

- `recover()` ngoài defer → luôn `nil`.  
- Sau recover, hàm chứa defer tiếp tục từ sau lần panic (thường return).

---

## 10. `defer` + recover patterns

**HTTP middleware / worker boundary** — không để panic hạ cả process:

```go
func recoverMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		defer func() {
			if rec := recover(); rec != nil {
				log.Printf("panic: %v\n%s", rec, debug.Stack())
				http.Error(w, "internal error", 500)
			}
		}()
		next.ServeHTTP(w, r)
	})
}
```

**Must-pattern** (init / test):

```go
func MustCompile(expr string) *Regexp {
	r, err := Compile(expr)
	if err != nil {
		panic(err)
	}
	return r
}
```

Chỉ dùng khi lỗi = programmer error không thể xử lý tại runtime hợp lý.

---

## 11. Khi nào KHÔNG dùng panic

| Dùng `error` | Dùng `panic` |
|--------------|--------------|
| I/O, network, parse user input | Bug: index âm tự viết, invariant nội bộ |
| Business rule thất bại | Init dữ liệu cứng hỏng (`Must`) |
| API công khai thư viện | Hiếm khi trong library — ưu tiên error |

- **Không** dùng panic như exception Java/C# cho “FileNotFound”.  
- **Không** recover rồi im lặng nuốt mọi thứ không log.  
- Goroutine panic không recover → crash **cả program** (trừ khi có recover trong goroutine đó).

```go
go func() {
	defer func() { recover() }() // nếu cần cô lập
	doWork()
}()
```

---

## 12. Best practices & checklist

1. Luôn xử lý `err` — đừng `_ =` trừ khi cố ý và an toàn.  
2. Thêm context bằng `%w` / `fmt.Errorf`.  
3. So sánh bằng `errors.Is` / `As`, không `==` trên wrapped.  
4. Sentinel hoặc typed error cho nhánh điều khiển.  
5. Log **một lần** ở biên (handler); tầng dưới return error.  
6. `errors.Join` khi nhiều lỗi độc lập cùng lúc.  
7. Panic chỉ cho bug; recover ở biên goroutine/http.  
8. Tránh typed nil trong `return err`.

### Checklist call site

```text
□ err kiểm tra ngay sau call
□ wrap đủ ngữ cảnh (hàm, id, path)
□ nhánh đặc biệt dùng Is/As
□ không panic cho lỗi người dùng
□ test phủ ErrNotFound / wrap chain
```

---

### Cheat sheet

| API | Việc |
|-----|------|
| `errors.New` | sentinel / message cố định |
| `fmt.Errorf("%w")` | wrap 1.13+ |
| `errors.Is` | so khớp chain |
| `errors.As` | lấy typed trong chain |
| `errors.Unwrap` | 1 lớp |
| `errors.Join` | nhiều lỗi 1.20+ |
| `panic`/`recover` | bug / biên cô lập |
