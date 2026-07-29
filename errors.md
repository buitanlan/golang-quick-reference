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
  - [6. `errors.AsType` (Go 1.26+)](#6-errorsastype-go-126)
  - [7. Cây lỗi và thứ tự duyệt](#7-cây-lỗi-và-thứ-tự-duyệt)
  - [8. Sentinel errors](#8-sentinel-errors)
  - [9. Custom error types](#9-custom-error-types)
  - [10. `errors.Join` (Go 1.20+)](#10-errorsjoin-go-120)
  - [11. Sentinel vs typed vs opaque](#11-sentinel-vs-typed-vs-opaque)
  - [12. Biên API: wrap hay che lỗi](#12-biên-api-wrap-hay-che-lỗi)
  - [13. Panic \& recover](#13-panic--recover)
  - [14. `defer` + recover patterns](#14-defer--recover-patterns)
  - [15. Khi nào KHÔNG dùng panic](#15-khi-nào-không-dùng-panic)
  - [16. Log lỗi với `log/slog`](#16-log-lỗi-với-logslog)
  - [17. Test lỗi](#17-test-lỗi)
  - [18. Best practices \& checklist](#18-best-practices--checklist)

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
- `error` là interface nên giá trị của nó là cặp `(type, data)`. `err == nil` chỉ true khi **cả hai** word bằng 0; một pointer nil nhưng có kiểu động → interface **khác** `nil`. Xem [typesystem.md](typesystem.md) §11 cho layout runtime.

```go
type MyError struct{}

func (*MyError) Error() string { return "my" }

func bad() error {
	var e *MyError // nil
	return e       // interface (type=*MyError, data=nil) → err != nil!
}

func good() error {
	var e *MyError
	if e == nil {
		return nil // trả nil interface tường minh
	}
	return e
}
```

Kiểm chứng: `bad() == nil` in ra `false`.

Hai biến thể hay gặp của bẫy này:

- Biến `err` khai báo là kiểu cụ thể (`var err *MyError`) rồi `return err` ở cuối hàm — luôn non-nil.
- Struct kết quả có field `Err *MyError` rồi gán vào một field `error`.

Cách tránh: **luôn** khai báo biến trung gian là `error`, và chỉ để hàm dựng lỗi trả về `error`.

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

Mỗi lần gọi `errors.New` tạo một **giá trị mới**: hai lỗi cùng text vẫn `!=` nhau.

```go
var ErrA = errors.New("dup")
var ErrB = errors.New("dup")
// ErrA == ErrB → false
```

Đó chính là lý do sentinel phải là **biến package-level** dùng lại, không phải `errors.New` gọi tại chỗ mỗi lần.

**Chi phí cấp phát**: tạo error mới luôn tốn allocation khi giá trị escape ra ngoài. Đo bằng `testing.AllocsPerRun` (go1.26.5, gán vào biến global để buộc escape):

```text
errors.New("x"):          1
fmt.Errorf("x"):          1
fmt.Errorf("x %d", i):    2
fmt.Errorf("ctx: %w", e): 2
return sentinel:          0
```

Go 1.26 đã giảm cấp phát của `fmt.Errorf` khi format string **không có verb** để ngang với `errors.New`. Nếu error không escape (compiler thấy được), `errors.New` có thể còn 0 allocation nhờ escape analysis — xem [pointers.md](pointers.md) §7.

Trên hot path mà lỗi là chuyện thường (parse loop, `io.EOF`), cách rẻ nhất vẫn là trả về **sentinel đã tạo sẵn**: 0 allocation.

### Quy ước message

- Chữ **thường** ở đầu, **không** dấu câu kết thúc: message sẽ bị nối vào message khác khi wrap.
- Dạng `"<operation>: <chi tiết>"`; không lặp lại từ `"error"` / `"failed"` (call site tự biết đó là lỗi).
- Prefix tên package cho sentinel export ra ngoài: `errors.New("user: not found")`.
- Dùng `%q` cho dữ liệu do người dùng cung cấp để tránh chuỗi rỗng/khoảng trắng gây khó đọc.

```go
// Tốt
fmt.Errorf("parse config %q: %w", path, err)

// Tránh
fmt.Errorf("Error: Failed to parse the config file!: %v", err)
```

---

## 4. Wrapping với `%w` (Go 1.13+)

```go
if err != nil {
	return fmt.Errorf("load config: %w", err)
}
```

- `%w` gắn lỗi gốc để `errors.Is` / `As` / `AsType` / `Unwrap` đi xuyên.  
- Từ **Go 1.20** một `fmt.Errorf` có thể chứa **nhiều** `%w`; khi đó lỗi trả về có `Unwrap() []error` (xem mục 7 và 10).  
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

Nhiều `%w` trong một lời gọi:

```go
both := fmt.Errorf("two: %w and %w", e1, e2)
// both.Error() == "two: e1 and e2"
// errors.Is(both, e1) && errors.Is(both, e2) → true
// errors.Unwrap(both) → nil  (chỉ có Unwrap() []error, không có Unwrap() error)
```

`errors.Unwrap`:

```go
inner := errors.Unwrap(err) // lỗi được wrap, hoặc nil
```

`%w` với toán hạng không phải `error` không wrap được và `go vet` báo lỗi:

```text
fmt.Errorf format %w has arg 42 of wrong type int
```

Runtime in ra `oops: %!w(int=42)`.

---

## 5. `errors.Is` / `errors.As` / `Unwrap`

### `errors.Is(err, target)`

Đi theo chain; true nếu gặp lỗi **bằng** target (qua `==` hoặc method `Is`).

```go
if errors.Is(err, fs.ErrNotExist) {
	// file không tồn tại
}
```

`errors.Is` khớp theo hai cơ chế, theo thứ tự: so sánh `==` (nếu kiểu comparable), rồi gọi method `Is(error) bool` nếu lỗi có.

### `errors.As(err, &target)`

Tìm lỗi trong chain gán được vào kiểu `target` trỏ tới:

```go
var pathErr *fs.PathError
if errors.As(err, &pathErr) {
	fmt.Println(pathErr.Path, pathErr.Op)
}
```

`target` **phải** là pointer non-nil tới kiểu implement `error` hoặc tới interface — nếu không, `errors.As` **panic** lúc runtime:

```text
panic: errors: target must be a non-nil pointer
```

`go vet` bắt được dạng sai tĩnh:

```text
second argument to errors.As must be a non-nil pointer to either a type that implements error, or to any interface type
```

Vì phải cấp phát/khai báo biến target trước, `errors.As` khá lắm lời; từ Go 1.26 ưu tiên `errors.AsType` (mục 6).

### Không dùng `==` / type assert mù quáng trên wrapped error

```go
// Yếu: mất khi đã wrap
if err == io.EOF { }

// Đúng:
if errors.Is(err, io.EOF) { }
```

Ngoại lệ đáng biết: `io.Reader` theo hợp đồng trả về đúng `io.EOF` **không wrap**, nên `err == io.EOF` vẫn chạy — nhưng `errors.Is` là mặc định an toàn hơn.

### `errors.ErrUnsupported` (Go 1.21+)

Sentinel chung để báo “thao tác không được hỗ trợ trên nền tảng/loại đối tượng này”. Stdlib wrap nó (ví dụ `os.Chmod` trên hệ thống không hỗ trợ), nên code portable nên check bằng `Is`:

```go
if err := f.Chmod(0o644); errors.Is(err, errors.ErrUnsupported) {
	// bỏ qua, không phải lỗi thật
}
```

Tự implement API tùy chọn thì wrap lại sentinel này thay vì tạo sentinel riêng:

```go
func (d *Driver) Truncate(n int64) error {
	return fmt.Errorf("driver.Truncate: %w", errors.ErrUnsupported)
}
```

---

## 6. `errors.AsType` (Go 1.26+)

```go
func AsType[E error](err error) (E, bool)
```

Bản generic của `errors.As`: **type-safe**, nhanh hơn, và không cần biến target.

```go
if pe, ok := errors.AsType[*fs.PathError](err); ok {
	fmt.Println(pe.Op, pe.Path)
}
```

- Trả về `(zero value của E, false)` khi không khớp — với `E` là pointer type thì zero value là `nil`.
- Duyệt cây giống `As`: `err` trước, rồi depth-first qua `Unwrap() error` / `Unwrap() []error`.
- Vẫn tôn trọng method `As(any) bool` của lỗi tự viết.
- Sai kiểu → **lỗi biên dịch**, không còn panic runtime như `As` với target sai.

So sánh nhanh:


| Nhu cầu | Dùng |
| ------- | ---- |
| Lấy lỗi theo kiểu cụ thể, biết kiểu lúc compile | `errors.AsType[E]` (1.26+) |
| Kiểu target chỉ biết lúc runtime (`reflect`, plugin) | `errors.As` |
| Target là biến đã có sẵn cần ghi vào | `errors.As` |
| Code phải build với Go < 1.26 | `errors.As` |
| Chỉ cần biết “có phải lỗi này” | `errors.Is` |

`errors.AsType` cũng thay được type assertion trực tiếp — và đúng hơn, vì nó xuyên qua wrap:

```go
// Sai khi err đã bị wrap
if pe, ok := err.(*fs.PathError); ok { }

// Đúng
if pe, ok := errors.AsType[*fs.PathError](err); ok { }
```

Muốn tìm theo **interface** (ví dụ `net.Error`) cũng được, vì `E` chỉ cần thỏa `error`:

```go
if ne, ok := errors.AsType[net.Error](err); ok && ne.Timeout() {
	// retry
}
```

---

## 7. Cây lỗi và thứ tự duyệt

Một lỗi có thể wrap **một** lỗi (`Unwrap() error`) hoặc **nhiều** lỗi (`Unwrap() []error`), nên chain thực chất là một **cây**. `Is` / `As` / `AsType` duyệt cây theo **pre-order, depth-first**: xét chính nó, rồi lần lượt từng con và toàn bộ cây con của con đó.

```go
type multi struct {
	msg  string
	errs []error
}

func (m *multi) Error() string   { return m.msg }
func (m *multi) Unwrap() []error { return m.errs }
```

Với cây `root -> [ left -> [codeErr{1}], codeErr{2} ]`, `errors.AsType[*codeErr]` trả về `codeErr{1}` — con đầu tiên theo thứ tự depth-first, không phải `codeErr{2}`.

Hệ quả thực tế:

- Thứ tự lỗi con **có ý nghĩa**: lỗi “cụ thể nhất / quan trọng nhất” nên đứng trước nếu bạn tự viết `Unwrap() []error`.
- `errors.Unwrap` (một lớp) trả về `nil` cho multi-error — nó chỉ hiểu `Unwrap() error`. Đừng dùng `errors.Unwrap` để lặp thủ công qua cây.
- Hợp đồng: `Unwrap() []error` **không được** chứa phần tử `nil`; và một kiểu không nên implement cả hai dạng `Unwrap`.

Bốn method tùy chọn mà lỗi tự viết có thể implement:


| Method | Ai gọi | Dùng khi |
| ------ | ------ | -------- |
| `Unwrap() error` | `errors.Unwrap`, `Is`, `As`, `AsType` | wrap đúng một lỗi nguyên nhân |
| `Unwrap() []error` | `Is`, `As`, `AsType` | gộp nhiều lỗi (multi-error) |
| `Is(error) bool` | `errors.Is` | so khớp một **họ** lỗi, không chỉ identity |
| `As(any) bool` | `errors.As`, `errors.AsType` | tự điền target, ví dụ dịch sang kiểu khác |

Ví dụ `Is` cho một họ lỗi:

```go
type KindError struct{ Kind string }

func (k *KindError) Error() string { return "kind " + k.Kind }

func (k *KindError) Is(target error) bool {
	t, ok := target.(*KindError)
	return ok && t.Kind == k.Kind
}

// errors.Is(fmt.Errorf("w: %w", &KindError{"net"}), &KindError{"net"}) → true
```

---

## 8. Sentinel errors

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

Sentinel là phần **API công khai**: một khi export, việc bỏ nó đi là breaking change. Chỉ export sentinel cho những nhánh mà caller thực sự cần phân biệt.

Ưu tiên **wrap** sentinel để giữ ngữ cảnh thay vì trả về sentinel trần:

```go
func Find(id int) (User, error) {
	row, err := db.QueryRow(id)
	if errors.Is(err, sql.ErrNoRows) {
		return User{}, fmt.Errorf("user.Find id=%d: %w", id, ErrNotFound)
	}
	return scan(row)
}
```

Sentinel nên là kiểu không thể sửa từ bên ngoài. `errors.New` trả về `*errorString` unexported nên an toàn; đừng dùng biến kiểu struct export ra ngoài vì caller có thể ghi đè field.

---

## 9. Custom error types

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

`net.Error.Temporary()` đã **deprecated**; dùng `Timeout()` hoặc check sentinel cụ thể (`os.ErrDeadlineExceeded`, `context.DeadlineExceeded`).

Hoặc dùng sentinel + wrap thay vì quá nhiều type.

Lấy typed error ra khỏi chain:

```go
// Go 1.26+
if ve, ok := errors.AsType[*ValidationError](err); ok {
	fmt.Println(ve.Field)
}

// Mọi phiên bản 1.13+
var ve *ValidationError
if errors.As(err, &ve) {
	fmt.Println(ve.Field)
}
```

### Quy ước cho custom type

- Dùng **pointer receiver** (`*ValidationError`) và trả về `*ValidationError`: đảm bảo so sánh theo identity và tránh copy. Nếu dùng value receiver thì cả `T` và `*T` đều là `error`, dễ lệch khi `As`/`AsType`.
- `Error()` **không** được panic và không được nil-deref: nó sẽ bị gọi khi log/format ở những nơi bạn không kiểm soát.
- `Error()` không nên gọi `Unwrap()` rồi format lồng nhau vô hạn; format nguyên nhân bằng `%v` một lần là đủ.
- Nếu type có `Unwrap()`, `Error()` nên bao gồm message của lỗi con để chuỗi cuối cùng vẫn đọc được.
- Tránh nhúng (`embed`) một `error` interface vào struct: `Error()` sẽ panic khi field đó nil.

Cấu trúc thường dùng: một type mang thông tin + wrap nguyên nhân.

```go
type OpError struct {
	Op   string
	Code int
	Err  error
}

func (e *OpError) Error() string {
	return fmt.Sprintf("%s: code %d: %v", e.Op, e.Code, e.Err)
}

func (e *OpError) Unwrap() error { return e.Err }
```

---

## 10. `errors.Join` (Go 1.20+)

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
- Trả `nil` nếu mọi input nil → gọi trực tiếp `return errors.Join(errs...)` là an toàn, không cần check `len(errs)`.
- `errors.Is` / `As` / `AsType` đi qua **tất cả** lỗi con.  
- Kết quả implement `Unwrap() []error` (multi-unwrap), **không** implement `Unwrap() error` → `errors.Unwrap` trả `nil`.
- `Error()` của kết quả nối các message bằng **newline** (`\n`), không phải `"; "`.

```go
err := errors.Join(ErrA, ErrB)
fmt.Println(errors.Is(err, ErrA)) // true
fmt.Println(errors.Is(err, ErrB)) // true
fmt.Printf("%q\n", err.Error())   // "e1\ne2" (dạng chung: mỗi lỗi một dòng)
```

Vì message có newline, `errors.Join` phù hợp để **báo cho người** (validation, shutdown) hơn là nhồi vào một dòng log; muốn log một dòng thì lặp qua `Unwrap() []error` và ghi từng attr (mục 16).

Kết hợp với wrap:

```go
return fmt.Errorf("shutdown: %w", errors.Join(errClose, errFlush))
```

Pattern tích lũy lỗi trong vòng lặp mà vẫn chạy hết:

```go
func CloseAll(cs []io.Closer) error {
	var errs []error
	for i, c := range cs {
		if err := c.Close(); err != nil {
			errs = append(errs, fmt.Errorf("close[%d]: %w", i, err))
		}
	}
	return errors.Join(errs...) // nil nếu không có lỗi nào
}
```

`errors.Join` khác `fmt.Errorf` nhiều `%w` ở chỗ: `Join` không thêm ngữ cảnh, `Errorf` cho bạn viết message. Chọn `Errorf` khi các lỗi có quan hệ ngữ nghĩa với nhau trong một câu, chọn `Join` khi chúng chỉ là một danh sách.

---

## 11. Sentinel vs typed vs opaque

Ba cách để caller “biết chuyện gì xảy ra”:


| Chiến lược | Caller kiểm tra bằng | Cho biết | Chi phí API |
| ---------- | -------------------- | -------- | ----------- |
| **Opaque** (mặc định) | chỉ `err != nil` | có lỗi | thấp nhất, tự do đổi bên trong |
| **Sentinel** | `errors.Is(err, pkg.ErrX)` | thuộc loại nào | phải giữ biến `ErrX` mãi |
| **Typed** | `errors.AsType[*pkg.XErr]` | loại + dữ liệu kèm theo | phải giữ cả struct + field |
| **Behaviour** | assert interface (`interface{ Timeout() bool }`) | khả năng, không phải danh tính | interface không cần export |

Nguyên tắc chọn:

- Mặc định là **opaque**: trả `error` đã wrap ngữ cảnh, không hứa gì thêm. Đây là dạng dễ tiến hóa nhất.
- Lên **sentinel** khi caller cần rẽ nhánh (`ErrNotFound` → 404, `ErrConflict` → 409) mà không cần dữ liệu kèm.
- Lên **typed** khi caller cần **field** (`ValidationError.Field`, `RetryError.After`, HTTP status code).
- **Behaviour** khi nhiều package độc lập cần cùng khả năng: định nghĩa interface ở phía **consumer**, không bắt producer import gì.

```go
// Phía consumer định nghĩa cái nó cần
type retryable interface{ Retryable() bool }

func shouldRetry(err error) bool {
	r, ok := errors.AsType[retryable](err)
	return ok && r.Retryable()
}
```

Đừng phát minh hierarchy lỗi kiểu Java (base class, hàng chục subclass). Trong Go, một sentinel + một type mang field thường phủ hết nhu cầu.

---

## 12. Biên API: wrap hay che lỗi

`%w` biến lỗi bên trong thành **một phần của hợp đồng công khai**: caller có thể `errors.Is` xuống tận đáy. Đó là điều bạn muốn đôi khi, và là rò rỉ abstraction những lúc khác.

**Wrap bằng `%w`** khi lỗi gốc thuộc contract mà bạn muốn giữ:

```go
// Người dùng có quyền biết đây là "file không tồn tại"
func LoadConfig(path string) (*Config, error) {
	b, err := os.ReadFile(path)
	if err != nil {
		return nil, fmt.Errorf("load config: %w", err) // fs.ErrNotExist đi xuyên
	}
	...
}
```

**Che bằng `%v`** (hoặc dịch sang sentinel của mình) khi lỗi gốc là chi tiết cài đặt:

```go
// Repo dùng Postgres hôm nay, có thể đổi mai — không để *pgconn.PgError lộ ra
func (r *Repo) Get(ctx context.Context, id int64) (User, error) {
	u, err := r.q.GetUser(ctx, id)
	switch {
	case errors.Is(err, sql.ErrNoRows):
		return User{}, fmt.Errorf("repo.Get id=%d: %w", id, ErrNotFound)
	case err != nil:
		return User{}, fmt.Errorf("repo.Get id=%d: %v", id, err) // không wrap
	}
	return u, nil
}
```

Checklist khi wrap:

- Mỗi tầng thêm **ngữ cảnh mới**, không lặp lại cái tầng dưới đã nói. Nếu message thành `"get user: get user: get user: ..."` là đang wrap thừa.
- Không nhét `err.Error()` vào message rồi lại `%w` cùng lỗi đó — sẽ in hai lần.
- Không wrap ở cả nơi trả về **và** nơi log; log một lần ở biên ngoài cùng.
- Với lỗi có nghĩa cho người dùng cuối (HTTP body, CLI stderr), map sang message riêng ở biên; đừng đẩy nguyên chain kỹ thuật ra ngoài.
- `context.Canceled` / `context.DeadlineExceeded` nên được wrap để đi xuyên (caller thường cần `errors.Is`) — xem [context.md](context.md).

---

## 13. Panic & recover

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
- Sau recover, hàm chứa defer **return ngay** (không tiếp tục thân hàm); các defer còn lại của hàm đó vẫn chạy.
- `recover` chỉ chặn được panic của **chính goroutine** đang chạy nó. Không có cách nào recover panic của goroutine khác từ bên ngoài.

### Ranh giới của `recover`

`recover()` chỉ trả về non-nil khi được gọi **trực tiếp bởi hàm deferred** của frame đang panic. Hàm deferred có tên cũng được:

```go
func helper() {
	if r := recover(); r != nil {
		fmt.Println("helper recovered:", r) // CHẠY
	}
}

func direct() {
	defer helper() // helper là hàm deferred → recover có hiệu lực
	panic("boom1")
}
```

Nhưng gọi `recover()` sâu thêm **một tầng** thì không:

```go
func inner() { fmt.Println("inner:", recover()) } // luôn nil

func nested() {
	defer func() {
		inner()                       // in "inner: <nil>"
		fmt.Println("after inner")    // vẫn chạy
	}()
	panic("boom2") // panic tiếp tục lan lên caller
}
```

Thứ tự sau khi recover thành công: hàm chứa defer đó return với các named result hiện tại, rồi các defer **còn lại** của cùng hàm vẫn chạy bình thường. Xem thêm [functions.md](functions.md) về `defer`.

### Đổi panic thành error ở biên

Dùng named return để chuyển panic thành `error` một cách tường minh:

```go
func Parse(b []byte) (v Value, err error) {
	defer func() {
		if r := recover(); r != nil {
			if e, ok := r.(error); ok {
				err = fmt.Errorf("parse: %w", e)
				return
			}
			err = fmt.Errorf("parse: %v", r)
		}
	}()
	return parseUnsafe(b), nil
}
```

Đừng recover những panic báo hiệu **bug thật sự** rồi tiếp tục như không có gì: `runtime.Error` (nil deref, index out of range) thường nên để crash trong service nội bộ, hoặc chí ít phải log kèm `debug.Stack()`.

```go
// Cân nhắc: panic của runtime thì re-panic
if re, ok := r.(runtime.Error); ok {
	panic(re)
}
```

### `panic(nil)` (thay đổi ở Go 1.21)

Từ Go 1.21, `panic(nil)` khiến `recover()` trả về `*runtime.PanicNilError` thay vì `nil`:

```go
defer func() {
	r := recover() // Go 1.21+: *runtime.PanicNilError
	fmt.Println(r) // "panic called with nil argument"
}()
panic(nil)
```

- Trước 1.21, `recover()` trả `nil` nên `if r := recover(); r != nil` **bỏ sót** panic này.
- Hành vi cũ có thể bật lại bằng `GODEBUG=panicnil=1`; hành vi mới phụ thuộc language version trong `go.mod`.
- Kết luận: đừng bao giờ `panic(nil)`; và code kiểm tra `r != nil` giờ đã đúng trong mọi trường hợp.

---

## 14. `defer` + recover patterns

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

**Cô lập worker goroutine** — panic trong goroutine **không** bị recover ở goroutine cha:

```go
func Go(fn func()) {
	go func() {
		defer func() {
			if r := recover(); r != nil {
				log.Printf("worker panic: %v\n%s", r, debug.Stack())
			}
		}()
		fn()
	}()
}
```

Nếu cần **chuyển** panic của goroutine con về cho cha, đưa qua channel hoặc `errgroup`:

```go
g, ctx := errgroup.WithContext(ctx)
g.Go(func() (err error) {
	defer func() {
		if r := recover(); r != nil {
			err = fmt.Errorf("worker panic: %v", r)
		}
	}()
	return work(ctx)
})
err := g.Wait()
```

Lưu ý: `net/http` đã tự recover panic của handler và đóng connection, nhưng nó **không** log stack theo format của bạn, và cũng không recover panic trong goroutine mà handler tự spawn. `http.ErrAbortHandler` là panic value đặc biệt: server bỏ qua nó không log — dùng khi muốn cắt response có chủ đích.

---

## 15. Khi nào KHÔNG dùng panic

| Dùng `error` | Dùng `panic` |
|--------------|--------------|
| I/O, network, parse user input | Bug: index âm tự viết, invariant nội bộ |
| Business rule thất bại | Init dữ liệu cứng hỏng (`Must`) |
| API công khai thư viện | Hiếm khi trong library — ưu tiên error |

- **Không** dùng panic như exception Java/C# cho “FileNotFound”.  
- **Không** recover rồi im lặng nuốt mọi thứ không log.  
- Goroutine panic không recover → crash **cả program** (trừ khi có recover trong goroutine đó).

Cô lập goroutine thì vẫn phải **log**, đừng nuốt trần:

```go
go func() {
	defer func() {
		if r := recover(); r != nil {
			log.Printf("panic: %v\n%s", r, debug.Stack())
		}
	}()
	doWork()
}()
```

---

## 16. Log lỗi với `log/slog`

`log/slog` (Go 1.21+) khuyến khích log lỗi thành **attribute** chứ không nhồi vào message.

```go
log := slog.New(slog.NewJSONHandler(os.Stdout, nil))
err := fmt.Errorf("save: %w", &OpError{Op: "insert", Code: 7, Err: os.ErrPermission})
log.Error("request failed", slog.Any("err", err))
```

Output thật (JSON handler):

```text
{"time":"...","level":"ERROR","msg":"request failed","err":"save: code 7: permission denied"}
```

`slog.Any` với một `error` render bằng `Error()` → ra **string**. Muốn log **có cấu trúc** thì implement `slog.LogValuer` trên error type:

```go
func (e *OpError) LogValue() slog.Value {
	return slog.GroupValue(
		slog.String("op", e.Op),
		slog.Int("code", e.Code),
		slog.Any("cause", e.Err),
	)
}
```

Output thật khi log trực tiếp giá trị đó:

```text
{"time":"...","level":"ERROR","msg":"with logvaluer","err":{"code":9,"cause":"file already closed"}}
```

Lưu ý: `LogValue` chỉ được dùng khi handler nhìn thấy chính giá trị đó. Nếu error đã bị `fmt.Errorf` wrap, attr là `*fmt.wrapError` — mất `LogValue`. Muốn giữ, lấy lỗi cụ thể ra trước bằng `errors.AsType` rồi log riêng.

Quy ước:

- Key thống nhất, thường là `"err"`; đừng lẫn `"error"`/`"err"`/`"reason"` trong cùng codebase.
- Log lỗi **một lần** ở biên ngoài cùng; tầng dưới chỉ wrap và return.
- Với multi-error, lặp qua `Unwrap() []error` và log thành slice để tránh newline trong một field.

```go
if u, ok := err.(interface{ Unwrap() []error }); ok {
	msgs := make([]string, 0, len(u.Unwrap()))
	for _, e := range u.Unwrap() {
		msgs = append(msgs, e.Error())
	}
	log.Error("batch failed", slog.Any("errs", msgs))
}
```

- Go 1.26 thêm `slog.NewMultiHandler` để fan-out cùng một record tới nhiều handler (ví dụ text ra stderr + JSON ra file).

---

## 17. Test lỗi

**Không** so sánh string. String message là chi tiết cài đặt và sẽ đổi khi bạn thêm ngữ cảnh wrap.

```go
// Giòn: vỡ ngay khi thêm một tầng wrap
if err.Error() != "user: not found" {
	t.Fatal("wrong error")
}

// Bền
if !errors.Is(err, user.ErrNotFound) {
	t.Fatalf("got %v, want ErrNotFound", err)
}
```

Kiểm tra typed error kèm field:

```go
func TestValidate(t *testing.T) {
	err := Validate(Input{Email: "x"})

	ve, ok := errors.AsType[*ValidationError](err) // Go 1.26+
	if !ok {
		t.Fatalf("got %T (%v), want *ValidationError", err, err)
	}
	if ve.Field != "Email" {
		t.Errorf("Field = %q, want %q", ve.Field, "Email")
	}
}
```

Table-driven với `wantErr` là sentinel:

```go
tests := []struct {
	name    string
	in      string
	wantErr error // nil = mong đợi thành công
}{
	{"ok", "a@b.c", nil},
	{"empty", "", ErrInvalidEmail},
}

for _, tt := range tests {
	t.Run(tt.name, func(t *testing.T) {
		_, err := Parse(tt.in)
		if !errors.Is(err, tt.wantErr) {
			t.Fatalf("err = %v, want %v", err, tt.wantErr)
		}
	})
}
```

`errors.Is(nil, nil)` là `true`, nên cùng một dòng check phủ được cả trường hợp thành công.

Test panic:

```go
func TestMustPanics(t *testing.T) {
	defer func() {
		r := recover()
		if r == nil {
			t.Fatal("want panic")
		}
	}()
	MustCompile("(")
}
```

Ghi chú thêm:

- Không assert vào **thứ tự** hay số lượng lỗi con của `errors.Join` nếu code sản xuất không đảm bảo thứ tự.
- Dùng `%v` / `%T` trong message của `t.Fatalf` để khi fail bạn thấy ngay lỗi thật là gì.
- `go vet` có sẵn check `errorsas`; chạy `go vet ./...` trong CI.

---

## 18. Best practices & checklist

1. Luôn xử lý `err` — đừng `_ =` trừ khi cố ý và an toàn.  
2. Thêm context bằng `%w` / `fmt.Errorf`; mỗi tầng một ngữ cảnh mới.  
3. So sánh bằng `errors.Is`; lấy kiểu bằng `errors.AsType` (1.26+) hoặc `errors.As`.  
4. Mặc định opaque; chỉ lên sentinel/typed khi caller thật sự cần rẽ nhánh.  
5. Không wrap `%w` lỗi thuộc chi tiết cài đặt qua biên API — dùng `%v` hoặc dịch sang sentinel của mình.  
6. Log **một lần** ở biên (handler); tầng dưới return error.  
7. `errors.Join` khi nhiều lỗi độc lập cùng lúc; nhớ message có newline.  
8. Panic chỉ cho bug; recover ở biên goroutine/http, luôn log `debug.Stack()`.  
9. Tránh typed nil trong `return err`.  
10. Test bằng `errors.Is`/`AsType`, không so string.

### Checklist call site

```text
□ err kiểm tra ngay sau call
□ wrap đủ ngữ cảnh (hàm, id, path)
□ nhánh đặc biệt dùng Is/AsType
□ %w chỉ khi lỗi gốc thuộc contract công khai
□ không panic cho lỗi người dùng
□ test phủ ErrNotFound / wrap chain
□ log 1 lần, key "err", không log lại ở tầng dưới
```

---

### Cheat sheet

| API | Việc |
|-----|------|
| `errors.New` | sentinel / message cố định |
| `fmt.Errorf("%w")` | wrap 1.13+; nhiều `%w` từ 1.20 |
| `errors.Is` | so khớp chain (identity hoặc method `Is`) |
| `errors.As` | lấy typed qua pointer target |
| `errors.AsType[E]` | lấy typed generic, 1.26+ |
| `errors.Unwrap` | 1 lớp, chỉ hiểu `Unwrap() error` |
| `errors.Join` | nhiều lỗi 1.20+, message cách nhau `\n` |
| `errors.ErrUnsupported` | “không hỗ trợ”, 1.21+ |
| `panic`/`recover` | bug / biên cô lập |
| `slog.Any("err", err)` | log lỗi có cấu trúc, 1.21+ |

### Tính năng theo phiên bản


| Version | Liên quan tới error |
| ------- | ------------------- |
| 1.13    | `errors.Is` / `As` / `Unwrap`, `%w` trong `fmt.Errorf` |
| 1.20    | `errors.Join`, nhiều `%w` trong một `fmt.Errorf`, `Unwrap() []error` |
| 1.21    | `errors.ErrUnsupported`, `panic(nil)` → `*runtime.PanicNilError`, `log/slog` |
| 1.26    | `errors.AsType[E]`, `fmt.Errorf` không format cấp phát ngang `errors.New`, `slog.NewMultiHandler` |
