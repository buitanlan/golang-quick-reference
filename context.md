# Context trong Go

`context.Context` mang **deadline, tín hiệu hủy, và giá trị request-scoped** xuyên suốt call chain. Mọi API I/O / RPC / DB hiện đại nên nhận `ctx` làm **tham số đầu tiên**.

---

## Mục lục

- [Context trong Go](#context-trong-go)
  - [Mục lục](#mục-lục)
  - [1. Interface `context.Context`](#1-interface-contextcontext)
  - [2. Tạo context gốc](#2-tạo-context-gốc)
  - [3. `WithCancel`](#3-withcancel)
  - [4. `WithTimeout` \& `WithDeadline`](#4-withtimeout--withdeadline)
  - [5. `WithValue`](#5-withvalue)
  - [6. Propagation \& cây context](#6-propagation--cây-context)
  - [7. `Done`, `Err`, `Deadline`](#7-done-err-deadline)
  - [8. Convention: `ctx` tham số đầu](#8-convention-ctx-tham-số-đầu)
  - [9. Request-scoped values — cảnh báo](#9-request-scoped-values--cảnh-báo)
  - [10. Patterns thực tế](#10-patterns-thực-tế)
  - [11. Pitfalls](#11-pitfalls)
  - [12. Best practices](#12-best-practices)

---

## 1. Interface `context.Context`

```go
type Context interface {
	Deadline() (deadline time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
	Value(key any) any
}
```

| Method | Ý nghĩa |
|--------|---------|
| `Deadline` | thời điểm hết hạn (nếu có) |
| `Done` | channel đóng khi hủy/hết hạn |
| `Err` | lý do sau khi Done đóng: `Canceled` / `DeadlineExceeded` |
| `Value` | lấy value theo key |

Package `context` (stdlib):

```go
import "context"
```

---

## 2. Tạo context gốc

```go
ctx := context.Background() // gốc không bao giờ hủy — main, init, top-level
ctx := context.TODO()       // chưa biết context nào — placeholder tạm
```

- `Background`: root trong `main`, test top-level, khi không có parent.  
- `TODO`: đánh dấu cần thay bằng ctx thật sau — tránh lan rộng trong production API.  
- Không bao giờ truyền `nil` Context — dùng `context.Background()` nếu thiếu.

```go
// BAD
DoWork(nil)

// GOOD
DoWork(context.Background())
```

---

## 3. `WithCancel`

```go
ctx, cancel := context.WithCancel(parent)
defer cancel() // luôn gọi để giải phóng resource cây con
```

Hủy thủ công:

```go
go func() {
	<-time.After(2 * time.Second)
	cancel()
}()

select {
case <-ctx.Done():
	return ctx.Err() // context.Canceled
case result := <-work:
	return result, nil
}
```

- `cancel` idempotent (gọi nhiều lần an toàn).  
- Sau cancel, mọi con derived cũng Done.  
- `WithCancelCause` (Go 1.20+): gắn cause tùy ý; lấy bằng `context.Cause(ctx)`.

```go
ctx, cancel := context.WithCancelCause(parent)
cancel(fmt.Errorf("client disconnected"))
fmt.Println(context.Cause(ctx))
```

---

## 4. `WithTimeout` & `WithDeadline`

```go
ctx, cancel := context.WithTimeout(parent, 5*time.Second)
defer cancel()

ctx2, cancel2 := context.WithDeadline(parent, time.Now().Add(5*time.Second))
defer cancel2()
```

- Timeout = Deadline = now + duration.  
- Hết hạn → `ctx.Err() == context.DeadlineExceeded`.  
- Vẫn `defer cancel()` để giải phóng sớm khi hàm return trước deadline.

```go
func fetch(ctx context.Context, url string) error {
	req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
	if err != nil {
		return err
	}
	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return err
	}
	defer resp.Body.Close()
	return nil
}
```

Lồng nhau: deadline con không thể **kéo dài** hơn parent; chỉ có thể chặt hơn.

---

## 5. `WithValue`

```go
type key int
const requestIDKey key = 0

ctx = context.WithValue(parent, requestIDKey, "abc-123")
id, _ := ctx.Value(requestIDKey).(string)
```

- Key nên kiểu **unexported** custom để tránh collision.  
- Value bất biến về mặt quy ước — không mutate object dùng chung thiếu sync.  
- Chỉ data **request-scoped** xuyên API (request ID, auth principal đã xác thực…).

```go
type ctxKey struct{}

func WithUser(ctx context.Context, u *User) context.Context {
	return context.WithValue(ctx, ctxKey{}, u)
}

func UserFrom(ctx context.Context) (*User, bool) {
	u, ok := ctx.Value(ctxKey{}).(*User)
	return u, ok
}
```

---

## 6. Propagation & cây context

```text
Background
 └── WithTimeout (5s)
      └── WithValue (requestID)
           └── WithCancel
                └── handler / DB / RPC
```

- Luôn truyền **cùng** hoặc **derived** ctx xuống callee.  
- Không lưu Context vào struct sống lâu hơn request (trừ khi field rõ ràng gắn lifetime request).  
- Hủy parent → mọi nhánh con nhận Done.

```go
func Handler(ctx context.Context) error {
	ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
	defer cancel()
	return service.Do(ctx)
}
```

---

## 7. `Done`, `Err`, `Deadline`

```go
select {
case <-ctx.Done():
	switch ctx.Err() {
	case context.Canceled:
		// hủy chủ động
	case context.DeadlineExceeded:
		// hết thời gian
	}
	return ctx.Err()
default:
	// vẫn còn hạn
}
```

Kiểm tra nhanh (không block):

```go
if err := ctx.Err(); err != nil {
	return err
}
```

Deadline:

```go
if dl, ok := ctx.Deadline(); ok {
	fmt.Println("deadline at", dl)
}
```

Worker loop:

```go
for {
	select {
	case <-ctx.Done():
		return ctx.Err()
	case job := <-jobs:
		if err := process(ctx, job); err != nil {
			return err
		}
	}
}
```

---

## 8. Convention: `ctx` tham số đầu

```go
func Query(ctx context.Context, db *sql.DB, q string) error
func (s *Server) Serve(ctx context.Context) error
```

- Tên thường `ctx`.  
- Không đặt sau optional args / trong giữa list.  
- Method: `ctx` vẫn đầu tiên (trước receiver không — sau receiver, đầu tiên trong params).  
- Không trả Context từ hàm trừ factory `WithXxx`.

Signature xấu:

```go
// BAD
func Query(q string, ctx context.Context)
func Query(ctx context.Context, opts ...Option, userID int) // dễ confuse
```

---

## 9. Request-scoped values — cảnh báo

**Được (hẹp):**

- Request ID / trace ID  
- Logger gắn request (cân nhắc)  
- User identity đã auth  

**Không dùng Context để:**

- Truyền tham số bắt buộc của hàm (dùng argument tường minh)  
- Optional config thông thường  
- Ẩn dependency (DB, logger global) thay DI rõ ràng  
- API công khai “ma thuật” khó test  

Lý do: Value không type-safe, không hiện trong signature, dễ quên document.

> Nếu thiếu value làm hàm sai → đó là **parameter**, không phải context value.

---

## 10. Patterns thực tế

**HTTP server** (stdlib đã gắn):

```go
func (h *Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context() // hủy khi client ngắt
	// ...
}
```

**Đợi nhiều việc + hủy:**

```go
ctx, cancel := context.WithCancel(context.Background())
defer cancel()

errCh := make(chan error, 1)
go func() { errCh <- work(ctx) }()

select {
case err := <-errCh:
	return err
case <-time.After(10 * time.Second):
	cancel()
	return context.DeadlineExceeded
}
```

**sql / http** tôn trọng ctx:

```go
row := db.QueryRowContext(ctx, "SELECT 1")
req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
```

**Ignore cancel chỉ khi thật sự cần** (hiếm — fire-and-forget detach):

```go
ctx = context.WithoutCancel(parent) // Go 1.21+
```

Dùng khi nhánh phải chạy xong dù request đã hủy (ví dụ ghi audit) — cân nhắc timeout riêng.

---

## 11. Pitfalls

1. Quên `defer cancel()` → leak timer/goroutine trong cây context.  
2. Truyền `context.Background()` xuống mọi tầng — mất hủy từ parent.  
3. Lưu `ctx` trong struct singleton sống suốt process.  
4. Dùng Value cho tham số bắt buộc.  
5. Không kiểm tra `ctx.Done()` trong loop dài / compute thuần.  
6. So sánh `err == context.Canceled` thay `errors.Is` khi đã wrap.  
7. Tạo timeout lồng nhau vô tội vạ mỗi helper — chồng deadline khó hiểu.

```go
if errors.Is(err, context.Canceled) { ... }
if errors.Is(err, context.DeadlineExceeded) { ... }
```

---

## 12. Best practices

1. `ctx` luôn tham số đầu; không nil.  
2. `defer cancel()` ngay sau `WithCancel`/`Timeout`/`Deadline`.  
3. Propagate ctx xuống mọi call I/O.  
4. Value chỉ request-scoped, key kiểu riêng.  
5. Tôn trọng hủy: `select` / API `*Context`.  
6. `WithoutCancel` hiếm và có timeout riêng.  
7. Test path hủy và path timeout.

### Cheat sheet

| API | Việc |
|-----|------|
| `Background` / `TODO` | gốc |
| `WithCancel` | hủy tay |
| `WithCancelCause` | hủy + cause (1.20+) |
| `WithTimeout` | hết hạn sau duration |
| `WithDeadline` | hết hạn tại thời điểm |
| `WithValue` | request value |
| `WithoutCancel` | tách hủy (1.21+) |
| `Done` / `Err` | tín hiệu & lý do |
| `Cause` | lý do tùy chỉnh (1.20+) |

---

### Checklist

```text
□ Hàm có I/O nhận ctx đầu tiên
□ cancel được defer
□ Client disconnect / timeout được kiểm tra
□ Không nhét business param vào Value
□ errors.Is với Canceled / DeadlineExceeded
```
