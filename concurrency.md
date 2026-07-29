# Goroutine & sync trong Go

Goroutine là đơn vị concurrency nhẹ của Go. Package `sync` cung cấp primitive đồng bộ (WaitGroup, Mutex, Once…). Channel (file riêng) phối hợp truyền dữ liệu; `sync` phối hợp chia sẻ bộ nhớ — **prefer channel khi truyền ownership**, mutex khi bảo vệ state ngắn.

---

## Mục lục

- [Goroutine \& sync trong Go](#goroutine--sync-trong-go)
  - [Mục lục](#mục-lục)
  - [1. Câu lệnh `go`](#1-câu-lệnh-go)
  - [2. Vòng đời goroutine](#2-vòng-đời-goroutine)
  - [3. `WaitGroup`](#3-waitgroup)
  - [4. `Mutex` \& `RWMutex`](#4-mutex--rwmutex)
  - [5. `Once`](#5-once)
  - [6. `Cond`](#6-cond)
  - [7. `sync/atomic`](#7-syncatomic)
  - [8. Race detector](#8-race-detector)
  - [9. Memory model (cơ bản)](#9-memory-model-cơ-bản)
  - [10. Pitfall: loop variable capture (\& Go 1.22)](#10-pitfall-loop-variable-capture--go-122)
  - [11. Worker pool (overview)](#11-worker-pool-overview)
  - [12. Best practices](#12-best-practices)

---

## 1. Câu lệnh `go`

```go
go fmt.Println("hello")

go func(id int) {
	fmt.Println("worker", id)
}(42)
```

- `go` khởi chạy hàm trên goroutine mới; caller **không** đợi.  
- Argument đánh giá **trước** khi goroutine chạy.  
- Không có handle ID chính thức — hủy qua `context` / channel `done`.

```go
go doWork(ctx) // truyền ctx để hủy
```

---

## 2. Vòng đời goroutine

1. Tạo → runnable.  
2. Scheduler M:N multiplex goroutine lên OS thread.  
3. Kết thúc khi hàm return.  
4. Panic không recover → **crash cả process**.  
5. Main return → process thoát; goroutine khác bị cắt (không join tự động).

```go
func main() {
	go work()
	// cần đợi: WaitGroup, channel, hoặc time.Sleep (chỉ demo)
}
```

Leak phổ biến: goroutine block mãi trên channel/mutex không bao giờ được signal.

---

## 3. `WaitGroup`

Đợi một nhóm goroutine xong:

```go
var wg sync.WaitGroup
for i := 0; i < 5; i++ {
	wg.Add(1)
	go func(i int) {
		defer wg.Done()
		work(i)
	}(i)
}
wg.Wait()
```

Quy tắc:

- `Add` **trước** `go`, hoặc `Add(n)` một lần trước vòng lặp.  
- Không copy `WaitGroup` sau khi dùng — truyền `*sync.WaitGroup`.  
- `Done` = `Add(-1)`; thường `defer wg.Done()`.  
- `Add` âm quá → panic.

---

## 4. `Mutex` & `RWMutex`

```go
type Counter struct {
	mu sync.Mutex
	n  int
}

func (c *Counter) Inc() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.n++
}

func (c *Counter) Value() int {
	c.mu.Lock()
	defer c.mu.Unlock()
	return c.n
}
```

**RWMutex** — nhiều reader hoặc một writer:

```go
var mu sync.RWMutex
var data map[string]int

func get(k string) (int, bool) {
	mu.RLock()
	defer mu.RUnlock()
	v, ok := data[k]
	return v, ok
}

func set(k string, v int) {
	mu.Lock()
	defer mu.Unlock()
	data[k] = v
}
```

- Giữ lock **ngắn**; không gọi I/O chậm trong critical section nếu tránh được.  
- Không copy struct chứa Mutex.  
- Deadlock: Lock hai lần cùng goroutine (Mutex không reentrant).

---

## 5. `Once`

Chạy đúng một lần (lazy init an toàn):

```go
var (
	once sync.Once
	cfg  *Config
)

func GetConfig() *Config {
	once.Do(func() {
		cfg = loadConfig()
	})
	return cfg
}
```

- Nếu `Do` panic, lần sau `Do` coi như đã chạy (không retry) — cẩn thận.  
- Dùng thay `init()` khi khởi tạo đắt / có thể thất bại cần handle khác.

---

## 6. `Cond`

Condition variable trên Mutex — đợi điều kiện phức tạp:

```go
var (
	mu      sync.Mutex
	cond    = sync.NewCond(&mu)
	ready   bool
)

// waiter
mu.Lock()
for !ready {
	cond.Wait() // Unlock mu, đợi; khi wake Lock lại
}
mu.Unlock()

// signaler
mu.Lock()
ready = true
cond.Signal() // hoặc Broadcast()
mu.Unlock()
```

- Luôn `Wait` trong vòng `for` kiểm tra điều kiện (spurious wake).  
- Nhiều case dùng channel rõ hơn Cond — Cond khi cần performance / pattern cổ điển.

---

## 7. `sync/atomic`

Thao tác không khóa trên số / pointer:

```go
var n atomic.Int64 // Go 1.19+ typed atomics
n.Add(1)
v := n.Load()

var flag atomic.Bool
flag.Store(true)
```

Cũ hơn:

```go
var n int64
atomic.AddInt64(&n, 1)
atomic.LoadInt64(&n)
```

- Đủ cho counter/flag đơn giản; cấu trúc phức tạp → Mutex.  
- `atomic.Value` lưu `any` snapshot (immutable-ish publish).

```go
var cfg atomic.Value
cfg.Store(&Config{Addr: ":8080"})
c := cfg.Load().(*Config)
```

---

## 8. Race detector

```bash
go test -race ./...
go run -race .
```

- Phát hiện concurrent read/write không đồng bộ cùng biến.  
- Chậm hơn ~2–20×, dùng RAM nhiều — CI / pre-release.  
- Không thay thế thiết kế đúng; chỉ bắt race thực sự xảy ra trong lần chạy.

Ví dụ race:

```go
// BAD
var n int
go func() { n++ }()
fmt.Println(n)
```

---

## 9. Memory model (cơ bản)

Happen-before đảm bảo visibility:

- Unlock mutex happen-before Lock tiếp theo thấy ghi trước Unlock.  
- Channel send happen-before receive tương ứng hoàn tất.  
- `go` statement happen-before start goroutine.  
- `Once.Do` hoàn tất happen-before các `Do` khác return.  
- Atomic sync theo tài liệu `sync/atomic`.

**Không** có guarantee nếu hai goroutine đọc/ghi biến thường không qua sync — dù “thấy đúng” trên máy bạn.

Chi tiết: [Go Memory Model](https://go.dev/ref/mem).

---

## 10. Pitfall: loop variable capture (& Go 1.22)

**Trước Go 1.22** — mọi iteration dùng **cùng** biến `i` / `v`:

```go
for i := 0; i < 3; i++ {
	go func() {
		fmt.Println(i) // thường in 3 3 3
	}()
}
```

Cách sửa cổ điển — shadow / truyền tham số:

```go
for i := 0; i < 3; i++ {
	i := i
	go func() { fmt.Println(i) }()
	// hoặc: go func(i int) { fmt.Println(i) }(i)
}
```

**Go 1.22+**: mỗi iteration tạo biến loop **mới** — ví dụ trên in 0,1,2 (thứ tự không đảm bảo).  
Vẫn nên truyền tham số tường minh khi code cần chạy trên nhiều phiên bản Go.

`range` tương tự (pre-1.22):

```go
for _, v := range items {
	go func() { use(v) }() // cũ: cùng v
}
```

---

## 11. Worker pool (overview)

Giới hạn số goroutine xử lý job:

```go
func RunPool(jobs <-chan Job, n int) {
	var wg sync.WaitGroup
	for i := 0; i < n; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			for j := range jobs {
				process(j)
			}
		}()
	}
	wg.Wait()
}

jobs := make(chan Job, 100)
go func() {
	defer close(jobs)
	for _, j := range all {
		jobs <- j
	}
}()
RunPool(jobs, 8)
```

- Buffer channel = hàng đợi.  
- `close(jobs)` khi hết việc để worker thoát `range`.  
- Hủy giữa chừng: `select` với `ctx.Done()` — xem [channels.md](channels.md), [context.md](context.md).

---

## 12. Best practices

1. Đừng tạo goroutine không có chiến lược thoát (leak).  
2. `WaitGroup.Add` trước `go`.  
3. Prefer `defer Unlock`.  
4. Chạy `-race` trên test concurrent.  
5. Tránh chia sẻ mutable state — truyền message bằng channel khi phù hợp.  
6. Mutex trong struct → method pointer receiver, không copy.  
7. Panic trong goroutine phải recover tại biên nếu cần sống sót.  
8. Document Go version nếu phụ thuộc loop semantics 1.22.

### Cheat sheet

| Tool | Việc |
|------|------|
| `go f()` | chạy concurrent |
| `WaitGroup` | đợi nhóm xong |
| `Mutex` | critical section |
| `RWMutex` | đọc nhiều |
| `Once` | init một lần |
| `atomic` | counter/flag |
| `-race` | tìm data race |
| Pool | giới hạn worker |
