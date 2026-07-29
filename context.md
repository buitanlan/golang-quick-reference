# Context trong Go

`context.Context` mang **deadline, tín hiệu hủy, và giá trị request-scoped** xuyên suốt call chain. Mọi API I/O / RPC / DB hiện đại nên nhận `ctx` làm **tham số đầu tiên**.

---

## Mục lục

- [Context trong Go](#context-trong-go)
  - [Mục lục](#mục-lục)
  - [1. Interface `context.Context`](#1-interface-contextcontext)
  - [2. Tạo context gốc](#2-tạo-context-gốc)
  - [3. `WithCancel` \& `WithCancelCause`](#3-withcancel--withcancelcause)
  - [4. `WithTimeout` / `WithDeadline` \& Cause](#4-withtimeout--withdeadline--cause)
  - [5. `AfterFunc`](#5-afterfunc)
  - [6. `WithValue`](#6-withvalue)
  - [7. Propagation \& cây context](#7-propagation--cây-context)
  - [8. `Done`, `Err`, `Deadline`, `Cause`](#8-done-err-deadline-cause)
  - [9. Convention: `ctx` tham số đầu](#9-convention-ctx-tham-số-đầu)
  - [10. Request-scoped values — cảnh báo](#10-request-scoped-values--cảnh-báo)
  - [11. Patterns thực tế](#11-patterns-thực-tế)
  - [12. Pitfalls](#12-pitfalls)
  - [13. Best practices](#13-best-practices)

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

`Cause` **không** nằm trên interface — là hàm package-level `context.Cause(ctx)` (Go 1.20+).

---

## 2. Tạo context gốc

```go
ctx := context.Background() // gốc không bao giờ hủy — main, init, top-level
ctx := context.TODO()       // chưa biết context nào — placeholder tạm
```

- `Background`: root trong `main`, test top-level, khi không có parent.
- `TODO`: đánh dấu cần thay bằng ctx thật sau — tránh lan rộng trong production API.
- Không bao giờ truyền `nil` Context — dùng `context.Background()` / `TODO()` nếu thiếu.

```go
// BAD
DoWork(nil)

// GOOD
DoWork(context.Background())
```

---

## 3. `WithCancel` & `WithCancelCause`

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
- Quên gọi `cancel` → leak cho tới khi parent hủy (`go vet` analyzer `lostcancel` — §12).

### `WithCancelCause` (Go 1.20+)

```go
ctx, cancel := context.WithCancelCause(parent)
cancel(fmt.Errorf("client disconnected"))

fmt.Println(ctx.Err())         // context.Canceled
fmt.Println(context.Cause(ctx)) // client disconnected
```

Semantics (theo `go doc`):

| Gọi | `ctx.Err()` | `context.Cause(ctx)` |
|---|---|---|
| `cancel(err)` với `err != nil` | `Canceled` | `err` |
| `cancel(nil)` | `Canceled` | `Canceled` |
| chưa hủy | `nil` | `nil` |

- Cause **đầu tiên** trên ctx hoặc ancestor thắng — hủy sau không ghi đè.
- Dùng cause khi cần phân biệt *vì sao* hủy (client gone, admin action…) mà vẫn giữ `Err() == Canceled` cho code cũ.

---

## 4. `WithTimeout` / `WithDeadline` & Cause

```go
ctx, cancel := context.WithTimeout(parent, 5*time.Second)
defer cancel()

ctx2, cancel2 := context.WithDeadline(parent, time.Now().Add(5*time.Second))
defer cancel2()
```

- Timeout = Deadline tại `now + duration`.
- Hết hạn → `ctx.Err() == context.DeadlineExceeded` (và `errors.Is`).
- Vẫn `defer cancel()` để giải phóng sớm khi hàm return trước deadline.

**Shortest wins:** deadline/timeout con **không thể kéo dài** hơn parent — chỉ có thể chặt hơn. Parent hết hạn → mọi con Done theo parent.

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

### `WithDeadlineCause` / `WithTimeoutCause` (Go 1.21+)

```go
ctx, cancel := context.WithTimeoutCause(parent, 3*time.Second, fmt.Errorf("upstream slow"))
defer cancel()

<-ctx.Done()
// ctx.Err()           → DeadlineExceeded
// context.Cause(ctx)  → upstream slow   (nếu hết hạn theo timeout này)
```

- Hết hạn theo deadline/timeout này → `Err()` vẫn `DeadlineExceeded`, `Cause` là lỗi bạn truyền.
- `CancelFunc` trả về **không** gắn cause khi gọi tay (hủy sớm → thường `Cause` ≈ `Canceled`).
- Cặp với `WithCancelCause` khi cần cause cho cả hủy tay và hết hạn.

---

## 5. `AfterFunc`

Go 1.21+: đăng ký callback chạy trong **goroutine riêng** khi `ctx` bị hủy (hoặc chạy ngay nếu đã hủy):

```go
stop := context.AfterFunc(ctx, func() {
	// cleanup / metrics / hủy công việc phụ
})

// ...
if stop() {
	// kịp hủy đăng ký — f chưa chạy
} else {
	// ctx đã hủy và f đã/đang start, hoặc đã stop trước đó
}
```

- Nhiều `AfterFunc` trên cùng ctx **độc lập**.
- `stop` không đợi `f` xong — cần sync riêng nếu phải biết `f` hoàn tất.
- Thay pattern `go func() { <-ctx.Done(); ... }()` khi chỉ cần một shot cleanup.

---

## 6. `WithValue`

```go
type key int
const requestIDKey key = 0

ctx = context.WithValue(parent, requestIDKey, "abc-123")
id, _ := ctx.Value(requestIDKey).(string)
```

- Key nên kiểu **unexported** custom (thường `type ctxKey struct{}`) để tránh collision giữa package.
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

## 7. Propagation & cây context

```text
Background
 └── WithTimeout (5s)
      └── WithValue (requestID)
           └── WithCancel
                └── handler / DB / RPC
```

- Luôn truyền **cùng** hoặc **derived** ctx xuống callee.
- **Không** lưu Context vào struct sống lâu hơn request (blog Go: *context and structs*). Field `ctx` trên singleton / long-lived worker là anti-pattern — truyền tham số tường minh.
- Ngoại lệ hẹp: request object có lifetime = request (`http.Request` đã mang ctx riêng).
- Hủy parent → mọi nhánh con nhận Done.

```go
func Handler(ctx context.Context) error {
	ctx, cancel := context.WithTimeout(ctx, 3*time.Second)
	defer cancel()
	return service.Do(ctx)
}
```

---

## 8. `Done`, `Err`, `Deadline`, `Cause`

```go
select {
case <-ctx.Done():
	switch {
	case errors.Is(ctx.Err(), context.Canceled):
		// hủy chủ động (hoặc cause signal — xem NotifyContext)
	case errors.Is(ctx.Err(), context.DeadlineExceeded):
		// hết thời gian
	}
	return ctx.Err()
default:
	// vẫn còn hạn
}
```

- `ctx.Err()` trực tiếp thường là sentinel — `==` thường đủ; khi error **đã wrap** hoặc đến từ `Cause` đặc biệt → dùng `errors.Is`.
- `DeadlineExceeded` implement `Is`/`Temporary`/`Timeout` — ưu tiên `errors.Is`.

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

Cause (1.20+):

```go
if err := context.Cause(ctx); err != nil {
	// lý do chi tiết hơn Err(); nếu không ai set cause → bằng Err()
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

## 9. Convention: `ctx` tham số đầu

```go
func Query(ctx context.Context, db *sql.DB, q string) error
func (s *Server) Serve(ctx context.Context) error
```

- Tên thường `ctx`.
- Không đặt sau optional args / trong giữa list.
- Method: `ctx` vẫn tham số đầu tiên **sau receiver**.
- Không trả Context từ hàm trừ factory `WithXxx`.
- Không nhét `ctx` vào struct dài hạn (§7).

Signature xấu:

```go
// BAD
func Query(q string, ctx context.Context)
func Query(ctx context.Context, opts ...Option, userID int) // dễ confuse
```

---

## 10. Request-scoped values — cảnh báo

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

## 11. Patterns thực tế

**HTTP server** (stdlib đã gắn):

```go
func (h *Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context() // hủy khi client ngắt
	// ...
}
```

**Timeout rõ ràng thay `time.After` thủ công:**

```go
ctx, cancel := context.WithTimeout(parent, 10*time.Second)
defer cancel()

select {
case err := <-errCh:
	return err
case <-ctx.Done():
	return ctx.Err()
}
```

**sql / http** tôn trọng ctx:

```go
row := db.QueryRowContext(ctx, "SELECT 1")
req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
```

**`WithoutCancel` (Go 1.21+)** — detach hủy parent (hiếm):

```go
ctx = context.WithoutCancel(parent)
```

- Derived **không** hủy khi parent hủy; `Done()` là `nil`, `Err()`/`Cause`/`Deadline` rỗng.
- Dùng khi nhánh phải chạy xong dù request đã hủy (ví dụ ghi audit) — nên gắn **timeout riêng**.

**Graceful shutdown với `signal.NotifyContext`:**

```go
ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt)
defer stop()

if err := run(ctx); err != nil && !errors.Is(err, context.Canceled) {
	return err
}
```

Go **1.26**: `NotifyContext` hủy bằng `CancelCauseFunc`; khi hủy do **signal**, `context.Cause(ctx)` trả về lỗi mô tả signal (ví dụ `"interrupt signal received"`). `errors.Is(cause, context.Canceled)` vẫn true. Gọi `stop()` sớm → cause thường là `Canceled`. Chi tiết entrypoint: [main-function.md](main-function.md).

**`AfterFunc` + shutdown:**

```go
stopHook := context.AfterFunc(ctx, func() {
	log.Print("shutting down:", context.Cause(ctx))
})
defer stopHook()
```

---

## 12. Pitfalls

1. Quên `defer cancel()` → leak timer/goroutine trong cây context. `go vet` — analyzer **`lostcancel`**: cancel từ `WithCancel` / `WithTimeout` / `WithDeadline` / biến thể Cause phải được gọi trên mọi control-flow path.
2. Truyền `context.Background()` xuống mọi tầng — mất hủy từ parent.
3. Lưu `ctx` trong struct singleton sống suốt process.
4. Dùng Value cho tham số bắt buộc.
5. Không kiểm tra `ctx.Done()` trong loop dài / compute thuần.
6. So sánh `err == context.Canceled` khi error đã wrap — dùng `errors.Is`.
7. Tạo timeout lồng nhau vô tội vạ mỗi helper — chồng deadline khó hiểu (nhớ shortest-wins).
8. Nhầm `Cause` với `Err`: code phân nhánh timeout/cancel nên nhìn `Err()`; `Cause` cho chi tiết / telemetry.

```go
if errors.Is(err, context.Canceled) { ... }
if errors.Is(err, context.DeadlineExceeded) { ... }
```

---

## 13. Best practices

1. `ctx` luôn tham số đầu; không nil; không cất trong struct dài hạn.
2. `defer cancel()` ngay sau `WithCancel`/`Timeout`/`Deadline` (và biến thể Cause).
3. Propagate ctx xuống mọi call I/O; tôn trọng `Done`.
4. Value chỉ request-scoped, key kiểu riêng unexported.
5. Phân biệt `Err()` (Canceled vs DeadlineExceeded) bằng `errors.Is`; dùng `Cause` khi cần lý do chi tiết (1.20+ / signal 1.26).
6. `WithoutCancel` hiếm và có timeout riêng; `AfterFunc` cho cleanup one-shot.
7. Test path hủy và path timeout; chạy `go vet` để bắt `lostcancel`.

### Cheat sheet

| API | Việc | Version |
|-----|------|---------|
| `Background` / `TODO` | gốc | — |
| `WithCancel` | hủy tay | — |
| `WithCancelCause` / `Cause` | hủy + cause | 1.20 |
| `WithTimeout` / `WithDeadline` | hết hạn | — |
| `WithTimeoutCause` / `WithDeadlineCause` | hết hạn + cause | 1.21 |
| `AfterFunc` | callback khi Done | 1.21 |
| `WithoutCancel` | tách hủy parent | 1.21 |
| `WithValue` | request value | — |
| `signal.NotifyContext` | hủy theo OS signal; Cause mô tả signal | 1.16; Cause: **1.26** |
| `Done` / `Err` / `Deadline` | tín hiệu & lý do chuẩn | — |

### Version map

| Phiên bản | Thay đổi |
|---|---|
| 1.16 | `signal.NotifyContext` |
| 1.20 | `WithCancelCause`, `Cause`, `CancelCauseFunc` |
| 1.21 | `AfterFunc`, `WithoutCancel`, `WithDeadlineCause`, `WithTimeoutCause` |
| 1.26 | `NotifyContext` hủy kèm cause → `context.Cause` cho biết signal |

### Checklist

```text
□ Hàm có I/O nhận ctx đầu tiên
□ cancel được defer / mọi path (lostcancel sạch)
□ Client disconnect / timeout được kiểm tra
□ Không nhét business param vào Value; không store ctx trong singleton
□ errors.Is với Canceled / DeadlineExceeded; Cause khi cần chi tiết
□ Shutdown: NotifyContext + stop(); đọc Cause từ 1.26 nếu cần biết signal
```
