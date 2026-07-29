# Goroutine & sync trong Go

Goroutine là đơn vị concurrency nhẹ của Go. Package `sync` cung cấp primitive đồng bộ (WaitGroup, Mutex, Once…). Channel (file riêng) phối hợp truyền dữ liệu; `sync` phối hợp chia sẻ bộ nhớ — **prefer channel khi truyền ownership**, mutex khi bảo vệ state ngắn.

Tài liệu này nhắm **Go 1.26**; API mới hơn Go 1.21 được ghi rõ phiên bản. Xem thêm [channels.md](channels.md), [context.md](context.md), [typesystem.md](typesystem.md).

---

## Mục lục

- [Goroutine \& sync trong Go](#goroutine--sync-trong-go)
  - [Mục lục](#mục-lục)
  - [1. Câu lệnh `go`](#1-câu-lệnh-go)
  - [2. Vòng đời goroutine](#2-vòng-đời-goroutine)
  - [3. `WaitGroup`](#3-waitgroup)
  - [4. `errgroup` \& `semaphore`](#4-errgroup--semaphore)
  - [5. `Mutex` \& `RWMutex`](#5-mutex--rwmutex)
  - [6. `Once` \& one-shot helpers](#6-once--one-shot-helpers)
  - [7. `Cond`](#7-cond)
  - [8. `sync/atomic`](#8-syncatomic)
  - [9. Race detector](#9-race-detector)
  - [10. Memory model \& happens-before](#10-memory-model--happens-before)
  - [11. Goroutine leak \& cách phát hiện](#11-goroutine-leak--cách-phát-hiện)
  - [12. Test tất định với `testing/synctest`](#12-test-tất-định-với-testingsynctest)
  - [13. Scheduler, GOMAXPROCS \& metrics](#13-scheduler-gomaxprocs--metrics)
  - [14. Pitfall: loop variable capture (\& Go 1.22)](#14-pitfall-loop-variable-capture--go-122)
  - [15. Worker pool (overview)](#15-worker-pool-overview)
  - [16. Best practices](#16-best-practices)

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

Đợi một nhóm goroutine xong. **Go 1.25+** có `(*WaitGroup).Go` — vừa `Add(1)`, vừa `go`, vừa `Done()`:

```go
var wg sync.WaitGroup
for i := range 5 {
	wg.Go(func() { work(i) })
}
wg.Wait()
```

Cách cổ điển (vẫn đúng, cần cho code chạy trên Go < 1.25):

```go
var wg sync.WaitGroup
for i := range 5 {
	wg.Add(1)
	go func() {
		defer wg.Done()
		work(i)
	}()
}
wg.Wait()
```

| | `wg.Go(f)` | `wg.Add(1)` + `defer wg.Done()` |
|---|-----------|--------------------------------|
| Quên `Add`/`Done` | không thể | lỗi kinh điển |
| `f` panic | **cấm** — doc yêu cầu `f` không panic | `defer Done` vẫn chạy khi unwind |
| Recover tại biên | phải tự bọc `f` | bọc trong closure |
| Kết quả / error | không có — dùng `errgroup` | không có |

Quy tắc:

- `Add` **trước** `go`, hoặc `Add(n)` một lần trước vòng lặp; `Add` sau khi `Wait` đã chạy trên WaitGroup rỗng là race.
- Không copy `WaitGroup` sau khi dùng — truyền `*sync.WaitGroup`.
- `Done` = `Add(-1)`; counter xuống dưới 0 → panic.
- Tái dùng một WaitGroup cho nhiều đợt: các `Go`/`Add` mới phải xảy ra **sau** khi `Wait` trước đó return.
- Memory model: `f` return **synchronizes before** `Wait` return — mọi ghi trong task thấy được sau `Wait`.
- `f` panic mà không recover → crash process, `Wait` không bao giờ return.

Bọc recover khi task có thể panic:

```go
wg.Go(func() {
	defer func() {
		if r := recover(); r != nil {
			log.Printf("task panic: %v", r)
		}
	}()
	work(i)
})
```

---

## 4. `errgroup` & `semaphore`

`sync.WaitGroup` không mang error và không hủy. `golang.org/x/sync/errgroup` (không phải stdlib) làm cả hai:

```go
import "golang.org/x/sync/errgroup"

func fetchAll(ctx context.Context, ids []int) ([]Item, error) {
	g, ctx := errgroup.WithContext(ctx)
	g.SetLimit(8) // tối đa 8 goroutine song song
	out := make([]Item, len(ids))
	for i, id := range ids {
		g.Go(func() error {
			it, err := fetchOne(ctx, id)
			if err != nil {
				return err // hủy ctx, các task khác thấy ctx.Done()
			}
			out[i] = it // mỗi task ghi index riêng → không cần lock
			return nil
		})
	}
	if err := g.Wait(); err != nil {
		return nil, err
	}
	return out, nil
}
```

- `WithContext` trả ctx con; **error đầu tiên** khác nil sẽ cancel ctx đó, `Wait` trả error đó.
- Zero `Group` hợp lệ: không giới hạn goroutine, **không** cancel khi lỗi.
- `SetLimit(n)`: `Go` block khi đã có n task chạy; `TryGo` trả `false` thay vì block.
- Không tái dùng một `Group` cho hai task khác nhau.
- Ghi vào slice theo index cố định là an toàn; ghi chung map/slice append thì vẫn cần mutex.

`golang.org/x/sync/semaphore` cho weighted semaphore tôn trọng ctx (mỗi job "nặng" khác nhau):

```go
sem := semaphore.NewWeighted(int64(runtime.GOMAXPROCS(0)))
var wg sync.WaitGroup
for _, t := range tasks {
	if err := sem.Acquire(ctx, t.Cost); err != nil {
		break // ctx đã hủy
	}
	wg.Go(func() {
		defer sem.Release(t.Cost)
		run(t)
	})
}
wg.Wait()
```

- `Acquire` trả `ctx.Err()` khi ctx hủy trước lúc lấy được slot — khác hẳn buffered channel (block vô hạn).
- Trọng số đều bằng 1 → buffered channel `chan struct{}` đơn giản hơn; xem [channels.md](channels.md).

---

## 5. `Mutex` & `RWMutex`

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
- Không copy struct chứa Mutex (`go vet` báo qua analyzer `copylocks`).
- Deadlock: Lock hai lần cùng goroutine (Mutex không reentrant).
- Mutex **không** gắn với goroutine: goroutine A `Lock`, goroutine B `Unlock` là hợp lệ về kỹ thuật nhưng khó đọc.
- `Unlock` một lock chưa được lock là **fatal error** của runtime (ví dụ `sync: Unlock of unlocked RWMutex`) — `recover` không cứu được.

`TryLock` / `TryRLock` (Go 1.18+) — chỉ dùng khi thật sự có phương án B:

```go
if !mu.TryLock() {
	return ErrBusy // không đợi
}
defer mu.Unlock()
```

**RWMutex caveat** (nói rõ trong `go doc sync.RWMutex`):

- Khi có goroutine gọi `Lock` trong lúc lock đang do reader giữ, mọi `RLock` mới sẽ **block** để writer không bị bỏ đói. Doc kết luận: điều này **cấm recursive read-locking** — `RLock` lồng nhau trong cùng goroutine có thể deadlock.
- `RLock` không "upgrade" được thành `Lock`, `Lock` không "downgrade" được thành `RLock`.
- Với critical section rất ngắn, `RWMutex` thường **chậm hơn** `Mutex` vì bookkeeping; chỉ đổi sang `RWMutex` khi đã đo được contention đọc.
- `RLocker()` trả một `sync.Locker` mà `Lock`/`Unlock` gọi `RLock`/`RUnlock` — hữu ích khi API nhận `sync.Locker` (ví dụ `sync.NewCond`).

`sync.Map` chỉ đáng dùng cho hai ca doc nêu rõ: (1) key ghi một lần đọc nhiều (cache chỉ lớn dần), (2) nhiều goroutine đọc/ghi các **tập key rời nhau**. Ngoài hai ca đó, `map` + `Mutex` type-safe hơn và thường nhanh hơn.

### Mutex hay channel?

| Tình huống | Chọn |
|-----------|------|
| Bảo vệ vài field, critical section ngắn | `Mutex` |
| Counter / flag | `sync/atomic` |
| Chuyển **ownership** dữ liệu sang goroutine khác | channel |
| Hàng đợi công việc, backpressure | buffered channel |
| Broadcast "shutdown" | `close(done)` / `context` |
| Đợi n việc xong | `WaitGroup.Go` / `errgroup` |
| Publish snapshot config đọc rất nhiều | `atomic.Pointer[T]` |

Kinh nghiệm: nếu code chỉ *bảo vệ state*, mutex đơn giản hơn; nếu code *điều phối luồng dữ liệu*, channel đơn giản hơn.

---

## 6. `Once` & one-shot helpers

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
- Memory model: `f()` chạy xong **synchronizes before** mọi `Do` return — nên `cfg` đọc được an toàn.
- `Once` không copy được sau lần dùng đầu.

**Go 1.21+** có ba helper gọn hơn, tự mang state, dùng được làm biến package:

```go
var getConfig = sync.OnceValue(func() *Config { return loadConfig() })

var loadDB = sync.OnceValues(func() (*sql.DB, error) { return sql.Open("pgx", dsn) })

var closeAll = sync.OnceFunc(func() { pool.Close() })
```

| Helper | Signature | Dùng khi |
|--------|-----------|----------|
| `sync.OnceFunc(f func())` | `func()` | side effect một lần |
| `sync.OnceValue[T](f func() T)` | `func() T` | lazy singleton |
| `sync.OnceValues[T1,T2](f func() (T1,T2))` | `func() (T1, T2)` | lazy init trả `(value, error)` |

- Cả ba **cache cả panic**: nếu `f` panic, mọi lần gọi sau panic lại với cùng giá trị đó (khác `Once.Do`, vốn coi như đã chạy và không panic lại).
- Chúng thay được pattern `var once sync.Once; var v T; var err error` thường bị quên đồng bộ.
- Vẫn cần "retry được khi lỗi"? `Once`/`OnceValues` không phù hợp — tự viết vòng lặp có mutex.

---

## 7. `Cond`

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

- Luôn `Wait` trong vòng `for` kiểm tra điều kiện — `Wait` không tự kiểm tra gì cả.
- `Wait` phải được gọi khi đang giữ `c.L`; `Wait` nhả lock, chờ, rồi lock lại trước khi return.
- Memory model: `Signal`/`Broadcast` **synchronizes before** `Wait` mà nó đánh thức.
- Doc `sync.Cond` khuyên: phần lớn trường hợp channel rõ hơn — `Broadcast` ≈ `close(ch)`, `Signal` ≈ gửi một giá trị.
- Không có `WaitContext`: `Cond` **không** hủy được bằng `context`. Cần timeout/cancel → dùng channel.
- `Cond` không copy được sau lần dùng đầu.

---

## 8. `sync/atomic`

Thao tác không khóa trên số / pointer. **Go 1.19+** có typed atomics — luôn ưu tiên chúng:

```go
var (
	requests atomic.Int64
	shutdown atomic.Bool
	current  atomic.Pointer[Config]
)

requests.Add(1)
n := requests.Load()
shutdown.CompareAndSwap(false, true)
```

| Type (1.19+) | Method chính |
|--------------|--------------|
| `atomic.Bool` | `Load` `Store` `Swap` `CompareAndSwap` |
| `atomic.Int32` `Int64` `Uint32` `Uint64` | thêm `Add`, và `And` / `Or` (Go 1.23+) |
| `atomic.Pointer[T]` | `Load` `Store` `Swap` `CompareAndSwap` (type-safe) |
| `atomic.Value` | `any`, mọi `Store` phải **cùng concrete type** |

Vì sao typed atomics hơn hàm cũ `atomic.AddInt64(&n, 1)`:

- Không thể "quên" dùng atomic ở một chỗ đọc — field chỉ có method atomic.
- Alignment: trên ARM, 386 và 32-bit MIPS, caller phải tự lo 64-bit alignment cho các hàm atomic nguyên thủy; `atomic.Int64` / `atomic.Uint64` **tự align**.
- `atomic.Pointer[T]` không cần `unsafe.Pointer` và không cần type assertion như `atomic.Value`.
- Không copy được vô tình an toàn hơn: các type này "must not be copied after first use" và `go vet` `copylocks` bắt được.

Publish config bất biến, reader không cần lock (copy-on-write):

```go
var current atomic.Pointer[Config]

func Load() *Config { return current.Load() }

func Reload(c *Config) { current.Store(c) } // ghi *mới*, không mutate cũ
```

- Điều kiện đúng: `*Config` sau khi publish là **read-only**. Mutate field của nó = data race dù đã atomic.
- `atomic.Value` panic nếu `Store` hai concrete type khác nhau, và panic với `Store(nil)`.
- Memory model: mọi atomic op trong một program hành xử như thể chạy theo **một thứ tự sequentially consistent** (ngang `volatile` của Java, `seq_cst` của C++). Nếu op B thấy hiệu ứng của op A thì A synchronizes before B.
- Atomic **không** làm nhiều biến nhất quán với nhau. Cần invariant giữa 2+ field → Mutex.

---

## 9. Race detector

```powershell
go test -race ./...
go run -race .
go build -race
```

- Phát hiện **data race**: một write xảy ra đồng thời với read/write khác trên cùng vùng nhớ, mà không phải toàn bộ đều là atomic op.
- Chi phí theo tài liệu Go: chậm hơn khoảng 2–20×, RAM tăng nhiều lần — dùng ở CI / pre-release, không phải build production.
- Chỉ hỗ trợ một số platform (`go help build`): `darwin/amd64`, `darwin/arm64`, `freebsd/amd64`, `linux/amd64`, `linux/arm64`, `linux/ppc64le`, `linux/riscv64`, `windows/amd64`.
- **Chỉ bắt race thật sự xảy ra trong lần chạy đó** — không phải static analysis. Test phải thực sự chạy đồng thời (nhiều goroutine, `t.Parallel`, `-count`), nếu không race vẫn ẩn.
- `go test -race` đánh dấu test là fail khi có report. Biến môi trường `GORACE` tinh chỉnh hành vi runtime của detector (tài liệu tại [Data Race Detector](https://go.dev/doc/articles/race_detector)).

Ví dụ race:

```go
// BAD
var n int
go func() { n++ }()
fmt.Println(n)
```

Race detector không thấy race nào không có nghĩa code đúng — nó chỉ nói "lần chạy này không quan sát được race".

---

## 10. Memory model & happens-before

[Go Memory Model](https://go.dev/ref/mem) đảm bảo: chương trình **data-race-free** chạy như thể mọi goroutine được ghép tuần tự trên một CPU (DRF-SC). Có race thì mọi bảo đảm mất — spec chỉ hứa implementation được phép báo race và kết thúc chương trình.

Spec dùng thuật ngữ **synchronized before** (subset của *happens before*). Các cạnh đồng bộ đáng nhớ:

| Cạnh | Nội dung |
|------|----------|
| init | `init` của package import xong trước `init` package importer; mọi `init` xong trước `main.main` |
| `go` | câu lệnh `go` synchronizes before goroutine bắt đầu chạy |
| goroutine exit | **không** có bảo đảm nào — goroutine kết thúc không đồng bộ với gì cả |
| channel send | send synchronizes before **hoàn tất** receive tương ứng |
| channel close | `close` synchronizes before receive trả zero value vì đã đóng |
| channel unbuffered | receive synchronizes before **hoàn tất** send tương ứng (chiều ngược lại!) |
| channel buffered cap C | receive thứ *k* synchronizes before hoàn tất send thứ *k+C* |
| `Mutex` | `Unlock` lần *n* synchronizes before `Lock` lần *n+1* return |
| `RWMutex` | `RLock` thấy một `Unlock` trước đó; `RUnlock` tương ứng synchronizes before `Lock` kế tiếp |
| `TryLock` | thành công = như `Lock`; **thất bại không tạo cạnh đồng bộ nào** |
| `Once` | `f()` xong synchronizes before mọi `Do` return |
| `WaitGroup` | `f` return (qua `Go`) synchronizes before `Wait` return |
| `Cond` | `Signal`/`Broadcast` synchronizes before `Wait` được đánh thức |
| atomic | tất cả atomic op có một thứ tự sequentially consistent chung |

Hệ quả hay bị quên:

- **Goroutine exit không đồng bộ.** `go func(){ a = 1 }(); print(a)` không bảo đảm gì; spec còn nói compiler "hung hăng" được phép xóa cả câu lệnh `go`.
- **Busy-wait trên `bool` thường là sai**, không chỉ chậm: `for !done {}` có thể không bao giờ kết thúc và có thể thấy `done == true` mà chưa thấy các ghi trước đó.

```go
// BAD: không có cạnh đồng bộ nào
var done bool
var msg string
go func() { msg = "hi"; done = true }()
for !done { }
print(msg) // có thể in ""
```

- **Đọc/ghi lớn hơn một machine word không nguyên tử.** Race trên interface, map, slice, string có thể tạo cặp (pointer, len)/(pointer, type) không khớp → **memory corruption**, không chỉ giá trị sai.
- Publish qua pointer cũng cần đồng bộ: thấy `g != nil` không bảo đảm thấy `g.msg` đã khởi tạo.
- Buffered channel cap ≥ 1 **không** cho bảo đảm ngược chiều mà unbuffered cho: đổi `make(chan int)` thành `make(chan int, 1)` có thể phá đúng đắn của code.

---

## 11. Goroutine leak & cách phát hiện

Goroutine leak = goroutine block mãi trên channel/mutex/syscall không bao giờ được signal. Nó giữ stack, giữ mọi biến nó capture, và không GC được — leak goroutine thường kéo theo leak memory.

Các pattern leak kinh điển:

```go
// 1. Consumer bỏ đi sớm, producer block trên send mãi mãi
func gen(n int) <-chan int {
	out := make(chan int) // unbuffered
	go func() {
		defer close(out)
		for i := range n {
			out <- i // block vĩnh viễn nếu caller break
		}
	}()
	return out
}

for v := range gen(1000) {
	if v > 3 {
		break // LEAK: goroutine kẹt ở out <- 4
	}
}
```

Sửa: cho producer biết đường thoát (hoặc dùng iterator, mục 13 của [channels.md](channels.md)):

```go
func gen(ctx context.Context, n int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for i := range n {
			select {
			case out <- i:
			case <-ctx.Done():
				return
			}
		}
	}()
	return out
}
```

Các nguồn leak khác:

- Gửi vào channel không ai nhận nữa (worker đã return vì error).
- `for range ch` mà không ai `close(ch)`.
- Chờ `WaitGroup` mà một task không bao giờ gọi `Done` (nhánh return sớm trước `defer`).
- Quên `defer cancel()` — cây context giữ node con đến khi parent hủy; xem [context.md](context.md).
- `http.Response.Body` không `Close` → goroutine đọc của transport còn sống.

Phát hiện:

| Cách | Dùng khi |
|------|----------|
| `runtime.NumGoroutine()` trước/sau test | smoke check nhanh, hay flaky |
| `pprof.Lookup("goroutine").WriteTo(w, 1)` | in stack mọi goroutine đang sống |
| `net/http/pprof` → `/debug/pprof/goroutine?debug=2` | production, xem goroutine kẹt ở đâu |
| `go.uber.org/goleak` (third-party) | assert trong `TestMain` |
| `testing/synctest` (1.25) | `Test` báo deadlock khi cả bubble kẹt (mục 12) |
| metric `/sched/goroutines:goroutines` | dashboard số goroutine sống |

Đọc profile: goroutine leak hiện ra dưới dạng **nhiều goroutine cùng stack** dừng ở đúng một dòng `chan send` / `chan receive` / `select`. Cột thời gian block trong `debug=2` cho biết chúng kẹt bao lâu.

```go
import _ "net/http/pprof"

func init() {
	go func() { log.Println(http.ListenAndServe("localhost:6060", nil)) }()
}
```

---

## 12. Test tất định với `testing/synctest`

`testing/synctest` là **experiment ở Go 1.24** (bật qua `GOEXPERIMENT=synctest`) và **thành package chuẩn ở Go 1.25**. Trên go1.26 toàn bộ API chỉ có hai hàm:

```go
func Test(t *testing.T, f func(*testing.T))
func Wait()
```

`Test` chạy `f` trong một **bubble** cô lập:

- Trong bubble, package `time` dùng **fake clock** riêng, bắt đầu từ nửa đêm UTC 2000-01-01.
- Thời gian chỉ tiến khi **mọi** goroutine trong bubble đã *durably blocked*; lúc đó clock nhảy tới mốc gần nhất đánh thức được ai đó. Nghĩa là `time.Sleep(time.Hour)` xong ngay lập tức.
- Nếu mọi goroutine durably blocked mà không có mốc thời gian nào cứu được → **deadlock**, `Test` panic. Đây chính là cách bắt goroutine leak.
- `Test` đợi mọi goroutine sinh trong bubble kết thúc trước khi return.
- `Wait()` block đến khi mọi goroutine *khác* trong bubble durably blocked — dùng để "để mọi thứ chạy tới điểm nghỉ" rồi mới assert.

Test hết hạn TTL 10 giây mà chạy tức thời:

```go
func TestExpiry(t *testing.T) {
	synctest.Test(t, func(t *testing.T) {
		ctx, cancel := context.WithTimeout(t.Context(), 10*time.Second)
		defer cancel()

		time.Sleep(9 * time.Second)
		synctest.Wait()
		if err := ctx.Err(); err != nil {
			t.Fatalf("tại 9s: ctx.Err() = %v, want nil", err)
		}

		time.Sleep(2 * time.Second)
		synctest.Wait()
		if err := ctx.Err(); err != context.DeadlineExceeded {
			t.Fatalf("tại 11s: ctx.Err() = %v, want DeadlineExceeded", err)
		}
	})
}
```

**Durably blocked** (thời gian được phép tiến):

- send/receive block trên channel **tạo trong bubble**
- `select` mà **mọi** case là channel tạo trong bubble
- `sync.Cond.Wait`
- `sync.WaitGroup.Wait`, nếu `Add` được gọi trong bubble
- `time.Sleep`

**Không** durably blocked (có thể bị đánh thức từ ngoài bubble):

- đợi `sync.Mutex` / `sync.RWMutex`
- I/O mạng, đọc file
- syscall

Hệ quả thực tế:

- Không dùng loopback network trong bubble — goroutine kẹt ở I/O khiến bubble không bao giờ idle. Dùng `net.Pipe()` hoặc fake transport.
- Channel / `time.Timer` / `time.Ticker` tạo trong bubble mà bị dùng từ ngoài bubble → **panic**.
- `sync.WaitGroup` gắn với bubble ở lần `Add`/`Go` đầu. Biến package `var wg sync.WaitGroup` **không** gắn được vào bubble (giới hạn kỹ thuật) nên có thể không durably blocking; `var wg = new(sync.WaitGroup)` thì được.
- Cleanup của `runtime.AddCleanup` và finalizer chạy **ngoài** mọi bubble.

Dùng cho: timeout, retry/backoff, TTL cache, rate limiter, heartbeat — mọi thứ trước đây phải test bằng `time.Sleep` và số magic.

---

## 13. Scheduler, GOMAXPROCS & metrics

Runtime dùng scheduler M:N — goroutine (G) chạy trên logical processor (P), P được OS thread (M) thực thi. `GOMAXPROCS` = số P, tức số goroutine chạy **cùng lúc** (không tính goroutine đang trong syscall).

**Go 1.25 đổi default `GOMAXPROCS`** (chi tiết trong `go doc runtime.GOMAXPROCS`):

- Nếu biến môi trường `GOMAXPROCS` là số nguyên dương → dùng giá trị đó.
- Ngược lại runtime chọn min của: số logical CPU, số CPU trong affinity mask, và (trên Linux) **giới hạn CPU throughput theo cgroup** = quota/period (`cpu.max` ở cgroup v2). Không bao giờ đặt dưới 2 trừ khi máy thật sự có < 2 CPU. Quota không tròn thì làm tròn lên.
- Runtime **tự cập nhật** giá trị này khi số CPU, affinity mask, hoặc cgroup quota đổi — tối đa mỗi giây một lần.
- Gọi `runtime.GOMAXPROCS(n)` hoặc set biến môi trường sẽ **tắt** auto-update. `runtime.SetDefaultGOMAXPROCS()` (Go 1.25+) khôi phục default và auto-update.
- Tắt hẳn: `GODEBUG=containermaxprocs=0` (bỏ phần cgroup, quay về `runtime.NumCPU`) và/hoặc `GODEBUG=updatemaxprocs=0` (bỏ auto-update). Hai GODEBUG này là **default cho module khai báo language version ≤ go1.24** — nâng `go` directive trong `go.mod` lên `go 1.25`+ mới hưởng behaviour mới.

Ý nghĩa: container có `limits.cpu: 2` trên máy 64 core từ Go 1.25 chạy với `GOMAXPROCS=2` thay vì 64 — bớt hẳn CPU throttling và latency spike do quá nhiều P.

Quan sát runtime:

```go
runtime.NumGoroutine() // số goroutine đang sống
runtime.NumCPU()       // logical CPU (không đổi trong lifetime process)
runtime.GOMAXPROCS(0)  // đọc giá trị hiện tại, không thay đổi
```

`runtime/metrics` cho số liệu chi tiết hơn và ổn định hơn parse text:

```go
samples := []metrics.Sample{
	{Name: "/sched/goroutines:goroutines"},
	{Name: "/sched/gomaxprocs:threads"},
	{Name: "/sched/goroutines/running:goroutines"},
	{Name: "/sched/goroutines/runnable:goroutines"},
	{Name: "/sched/goroutines/waiting:goroutines"},
}
metrics.Read(samples)
for _, s := range samples {
	fmt.Println(s.Name, s.Value.Uint64())
}
```

| Metric (có trong go1.26) | Ý nghĩa |
|--------------------------|---------|
| `/sched/goroutines:goroutines` | số goroutine sống |
| `/sched/goroutines/running:goroutines` | đang thực thi; ≤ `/sched/gomaxprocs:threads` |
| `/sched/goroutines/runnable:goroutines` | sẵn sàng chạy nhưng chưa có P — **runnable cao kéo dài = thiếu CPU** |
| `/sched/goroutines/waiting:goroutines` | đang đợi resource (channel, lock, timer) |
| `/sched/goroutines/not-in-go:goroutines` | đang trong syscall / cgo |
| `/sched/goroutines-created:goroutines` | tổng goroutine đã tạo từ khi start |
| `/sched/gomaxprocs:threads` | `GOMAXPROCS` hiện tại (theo dõi auto-update 1.25) |
| `/sched/latencies:seconds` | histogram thời gian goroutine chờ trong scheduler |

Dùng `metrics.All()` để lấy danh sách kèm mô tả của toolchain đang chạy — tên metric có thể được thêm/bỏ giữa các phiên bản, nên luôn kiểm tra `Sample.Value.Kind() != metrics.KindBad` trước khi đọc.

---

## 14. Pitfall: loop variable capture (& Go 1.22)

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

**Go 1.22+**: mỗi iteration tạo biến loop **mới** — ví dụ trên in 0,1,2 (thứ tự không đảm bảo). `range` cũng vậy, nên `for _, v := range items { go func() { use(v) }() }` giờ đã đúng.

Chi tiết quan trọng: semantics này quyết định bởi **language version của file**, tức `go` directive trong `go.mod` (hoặc `//go:build go1.x` trong file), **không** phải version toolchain. Build một module `go 1.21` bằng go1.26 vẫn cho semantics cũ. Muốn soi code cũ: `go vet` có analyzer `loopclosure` — "check references to loop variables from within nested functions" (`-rangeloops` là alias deprecated của flag này).

Nâng `go.mod` lên `go 1.22`+ là cách sửa hàng loạt; sau đó `i := i` chỉ còn là noise. Vẫn giữ tham số tường minh nếu library phải build được với toolchain cũ.

---

## 15. Worker pool (overview)

Giới hạn số goroutine xử lý job:

```go
func RunPool(ctx context.Context, jobs <-chan Job, n int) {
	var wg sync.WaitGroup
	for range n {
		wg.Go(func() {
			for {
				select {
				case <-ctx.Done():
					return
				case j, ok := <-jobs:
					if !ok {
						return
					}
					process(j)
				}
			}
		})
	}
	wg.Wait()
}

jobs := make(chan Job, 100)
go func() {
	defer close(jobs)
	for _, j := range all {
		select {
		case jobs <- j:
		case <-ctx.Done():
			return
		}
	}
}()
RunPool(ctx, jobs, 8)
```

- Buffer channel = hàng đợi; `close(jobs)` khi hết việc để worker thoát.
- Producer cũng phải `select` với `ctx.Done()`, nếu không nó leak khi worker bỏ đi.
- Cần error + hủy tự động → `errgroup` (mục 4) ngắn hơn nhiều.
- Số worker: I/O-bound thì cao hơn `GOMAXPROCS` hợp lý; CPU-bound thì ≈ `runtime.GOMAXPROCS(0)`.
- Chi tiết pattern channel: [channels.md](channels.md); hủy: [context.md](context.md).

---

## 16. Best practices

1. Mỗi goroutine phải có **đường thoát rõ ràng** trước khi viết `go` — ai đóng channel, ctx nào hủy nó.
2. Ưu tiên `wg.Go(f)` (1.25+); dùng `errgroup` khi cần error/cancel.
3. Prefer `defer Unlock`; giữ critical section ngắn, không I/O trong đó.
4. Chạy `-race` trên test concurrent trong CI; test phải thực sự đồng thời.
5. Tránh chia sẻ mutable state — truyền ownership qua channel, hoặc publish snapshot bằng `atomic.Pointer[T]`.
6. Mutex trong struct → method pointer receiver, không copy (`go vet` analyzer `copylocks`).
7. Panic trong goroutine crash cả process — recover tại biên nếu goroutine đó không được phép giết service.
8. Không dùng `time.Sleep` để đồng bộ trong test; dùng `testing/synctest` (1.25+).
9. Đặt `go` directive trong `go.mod` ≥ `go 1.22` để hết pitfall loop variable, ≥ `go 1.25` để hưởng default `GOMAXPROCS` theo cgroup.
10. Đo trước khi tối ưu: `-race`, `-cpuprofile`, `/sched/latencies:seconds`, `/sched/goroutines/runnable:goroutines`.

### Cheat sheet

| Tool | Việc |
|------|------|
| `go f()` | chạy concurrent |
| `wg.Go(f)` | spawn + đợi (1.25+) |
| `WaitGroup` | đợi nhóm xong |
| `errgroup.Group` | đợi + error + cancel (x/sync) |
| `semaphore.Weighted` | giới hạn có ctx (x/sync) |
| `Mutex` | critical section |
| `RWMutex` | đọc nhiều, đã đo contention |
| `Once` / `OnceValue` | init một lần |
| `atomic.Int64` / `Pointer[T]` | counter / publish snapshot |
| `-race` | tìm data race |
| `synctest.Test` | test concurrency tất định |
| `NumGoroutine` / `runtime/metrics` | quan sát leak & scheduler |

### Version map

| Phiên bản | Thay đổi liên quan concurrency |
|-----------|-------------------------------|
| 1.18 | `Mutex.TryLock`, `RWMutex.TryRLock` |
| 1.19 | typed atomics: `atomic.Bool` `Int64` `Pointer[T]`… |
| 1.21 | `sync.OnceFunc` / `OnceValue` / `OnceValues`; `context.AfterFunc`, `WithoutCancel`, `WithDeadlineCause`, `WithTimeoutCause` |
| 1.22 | per-iteration loop variable (hết pitfall capture) |
| 1.23 | GC thu hồi được timer/ticker chưa `Stop`; timer channel thành unbuffered; `iter.Seq` + range-over-func |
| 1.24 | `runtime.AddCleanup`, `weak.Pointer`, `testing.T.Context`; `synctest` là experiment |
| 1.25 | `WaitGroup.Go`; `testing/synctest` chuẩn; default `GOMAXPROCS` theo cgroup + auto-update, `runtime.SetDefaultGOMAXPROCS` |
| 1.26 | `signal.NotifyContext` hủy kèm cause → `context.Cause(ctx)` cho biết signal nào |

### Checklist

```text
□ Mỗi goroutine có điều kiện thoát xác định
□ Không copy struct chứa Mutex/WaitGroup/Once/Cond
□ Invariant nhiều field → Mutex, không atomic rời rạc
□ ctx được truyền và tôn trọng ở mọi vòng lặp dài
□ Test concurrent chạy với -race
□ Test phụ thuộc thời gian dùng synctest, không Sleep
□ go.mod khai báo go 1.22+ (loop var) / 1.25+ (GOMAXPROCS)
```
