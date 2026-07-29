# Channel trong Go

Channel là primitive giao tiếp giữa goroutine: **gửi/nhận giá trị** và đồng bộ. “Don’t communicate by sharing memory; share memory by communicating.”

Tài liệu này nhắm **Go 1.26**. Xem thêm [concurrency.md](concurrency.md), [context.md](context.md), [typesystem.md](typesystem.md), [statements.md](statements.md).

---

## Mục lục

- [Channel trong Go](#channel-trong-go)
  - [Mục lục](#mục-lục)
  - [1. Tạo channel: `make(chan T)`](#1-tạo-channel-makechan-t)
  - [2. Unbuffered vs buffered](#2-unbuffered-vs-buffered)
  - [3. Send \& receive](#3-send--receive)
  - [4. `close` \& nhận từ channel đóng](#4-close--nhận-từ-channel-đóng)
  - [5. `range` trên channel](#5-range-trên-channel)
  - [6. `select`](#6-select)
  - [7. Nil channel behavior](#7-nil-channel-behavior)
  - [8. Unidirectional channel types](#8-unidirectional-channel-types)
  - [9. Patterns: fan-in/out, pipeline, done](#9-patterns-fan-inout-pipeline-done)
  - [10. Timer, ticker \& timeout](#10-timer-ticker--timeout)
  - [11. Hủy với `context` \& backpressure](#11-hủy-với-context--backpressure)
  - [12. Semaphore \& worker pool](#12-semaphore--worker-pool)
  - [13. Iterator thay generator channel](#13-iterator-thay-generator-channel)
  - [14. Channel of channels \& request/reply](#14-channel-of-channels--requestreply)
  - [15. Memory model \& chi phí channel](#15-memory-model--chi-phí-channel)
  - [16. Khi nào `close`](#16-khi-nào-close)
  - [17. Pitfalls](#17-pitfalls)
  - [18. Best practices](#18-best-practices)

---

## 1. Tạo channel: `make(chan T)`

```go
var c chan int           // nil channel
c = make(chan int)       // unbuffered
b := make(chan string, 8) // buffered capacity 8
```

- Zero value = `nil` (xem mục 7).  
- Kiểu `chan T` — T có thể là struct, pointer, `struct{}` (signal).  
- Channel là reference-like: gán/copy handle cùng queue.

```go
done := make(chan struct{})
```

---

## 2. Unbuffered vs buffered

| | Unbuffered `make(chan T)` | Buffered `make(chan T, n)` |
|---|---------------------------|----------------------------|
| Send | block đến khi có receiver | block khi đầy (len==cap) |
| Receive | block đến khi có sender | block khi rỗng |
| Đồng bộ | handshake đồng bộ chặt | hàng đợi n phần tử |

```go
ch := make(chan int)
go func() { ch <- 1 }() // phải ở goroutine khác: send unbuffered block đến khi có receiver
v := <-ch
```

Buffered:

```go
ch := make(chan int, 2)
ch <- 1
ch <- 2
// ch <- 3 // block: buffer đầy và chưa có ai nhận
```

**Khác biệt về bảo đảm đồng bộ, không chỉ về hiệu năng.** Với unbuffered channel, receive *synchronizes before* khi send hoàn tất — sender biết chắc receiver đã nhận. Buffered channel **không** cho bảo đảm đó: `ch <- v` return ngay khi giá trị vào buffer.

```go
// Unbuffered: in "hello, world" chắc chắn
var c = make(chan int)
var a string

func f() { a = "hello, world"; <-c }

func main() {
	go f()
	c <- 0    // return sau khi f() đã nhận
	print(a)  // thấy được a
}
```

Đổi `make(chan int)` thành `make(chan int, 1)` trong ví dụ trên là **bug**: `print(a)` có thể in chuỗi rỗng. Đây là lý do "tăng buffer cho nhanh" đôi khi phá đúng đắn. Xem mục 15 và [Go Memory Model](https://go.dev/ref/mem).

- Unbuffered = **rendezvous**: một điểm đồng bộ hai chiều, dùng để bàn giao ownership.
- Buffer 1 = mailbox / “signal không mất”; hợp cho `errCh := make(chan error, 1)` để goroutine gửi lỗi rồi thoát ngay dù caller đã bỏ đi.
- Buffer n = hàng đợi hấp thụ burst; **không** giải quyết vấn đề nếu producer nhanh hơn consumer về lâu dài — vẫn cần backpressure (mục 11).
- `len(ch)` / `cap(ch)` chỉ là snapshot; đừng dùng để quyết định logic ("nếu chưa đầy thì gửi" là race). Muốn thử gửi: `select` + `default`.

---

## 3. Send & receive

```go
ch <- v      // send
v := <-ch    // receive
v, ok := <-ch // ok=false nếu đóng và hết dữ liệu
```

- Send/receive trên nil → block mãi.  
- Send trên closed → **panic** `send on closed channel`.
- Receive trên closed → zero value + `ok=false` (sau khi drain).

```go
len(ch) // số phần tử đang trong buffer
cap(ch) // capacity
```

Bảng tổng hợp toàn bộ tổ hợp trạng thái — học bảng này là hiểu 90% bug channel:

| Thao tác | `nil` channel | open, sẵn sàng | open, chưa sẵn sàng | closed |
|----------|---------------|----------------|---------------------|--------|
| send `ch <- v` | block mãi | thành công | block | **panic** |
| receive `<-ch` | block mãi | trả giá trị | block | zero value ngay, `ok=false` |
| `close(ch)` | **panic** | thành công | thành công | **panic** |
| `len` / `cap` | `0` / `0` | số phần tử / cap | — | vẫn đọc được |
| case trong `select` | **không bao giờ** ready | ready | không ready | **luôn** ready |

Hai dòng quan trọng nhất: receive từ channel đã đóng **luôn ready ngay lập tức** (nên `case <-done:` trong `select` thắng ngay sau `close(done)`), và case trên nil channel **không bao giờ** được chọn (nên gán `nil` là cách tắt một nhánh `select`, mục 7).

Chi tiết đánh giá biểu thức: trong `ch <- f()`, cả `ch` và `f()` được đánh giá **trước** khi thao tác gửi block. Nếu `ch` là biểu thức có side effect, side effect đó xảy ra dù send có block hay panic.

Receive từ channel đã đóng không tiêu thụ gì cả — đọc bao nhiêu lần cũng trả zero value, nên `for { <-done }` là busy loop CPU 100%.

---

## 4. `close` & nhận từ channel đóng

```go
close(ch)
```

- Chỉ **người gửi** (hoặc coordinator) đóng — không phải mọi receiver.  
- Đóng hai lần → panic `close of closed channel`.
- Đóng nil channel → panic `close of nil channel`.
- Đóng channel chỉ-nhận (`<-chan T`) → **compile error**.
- Sau close: nhận hết buffered rồi `ok=false`.

```go
ch := make(chan int, 2)
ch <- 1
close(ch)
v1, ok1 := <-ch // 1, true
v2, ok2 := <-ch // 0, false
```

`close` **không** giải phóng channel và không cần thiết cho GC — channel không còn ai tham chiếu sẽ được GC dù chưa đóng. `close` chỉ có một nghĩa: **"không còn giá trị nào nữa"**. Vì thế:

- Không cần `close` mọi channel; chỉ đóng khi receiver cần biết "hết".
- `close` là **broadcast một lần**, không hủy được. Không có `reopen`.

Signal “xong” không cần payload — mọi waiter unblock cùng lúc, và mọi lần đọc sau đó vẫn ready ngay:

```go
close(done) // mọi waiter <-done unblock
```

Muốn "đóng an toàn nhiều lần" thì đừng bọc `close` bằng mutex — dùng `sync.Once` hoặc `context`:

```go
type Stopper struct{ once sync.Once; done chan struct{} }

func (s *Stopper) Stop() { s.once.Do(func() { close(s.done) }) }
```

Đây là lý do `context.Context` tồn tại: nó chính là "done channel + lý do + deadline + cây" đã đóng gói đúng. Xem [context.md](context.md).

---

## 5. `range` trên channel

```go
for v := range ch {
	process(v)
}
// thoát khi ch đóng và hết phần tử
```

Tương đương:

```go
for {
	v, ok := <-ch
	if !ok {
		break
	}
	process(v)
}
```

- Quên `close` → `range` block mãi (leak).  
- Không range nếu channel không bao giờ đóng — dùng `select` + done.
- `for range ch` **không** có nhánh hủy. Cần hủy giữa chừng → `select` (mục 11).
- `break` trong `for range ch` thoát loop nhưng **không** báo cho sender: sender đang block trên `ch <- v` sẽ leak. Đây là leak channel phổ biến nhất; xem mục 11 của [concurrency.md](concurrency.md).
- Chỉ range trên channel có **một** người đóng xác định. Nhiều sender → coordinator đóng (mục 16).
- `for range ch { }` (không nhận giá trị) hợp lệ — hữu ích khi chỉ cần đợi hoặc drain.

Drain phần còn lại để sender không kẹt (khi sender không biết hủy):

```go
go func() {
	for range ch {
	}
}()
```

Chỉ dùng khi biết chắc `ch` sẽ được đóng — nếu không đây lại là một leak khác.

---

## 6. `select`

Chờ nhiều thao tác channel; chọn nhánh **sẵn sàng**:

```go
select {
case v := <-in:
	fmt.Println("got", v)
case out <- result:
	fmt.Println("sent")
case <-ctx.Done():
	return ctx.Err()
case <-time.After(time.Second):
	return errors.New("timeout")
}
```

Semantics chính xác:

1. Mọi **channel operand** và mọi biểu thức giá trị bên phải `<-` của case send được đánh giá **đúng một lần**, theo thứ tự nguồn, **trước** khi chọn case. Nghĩa là `case <-time.After(d):` tạo timer mỗi lần `select` chạy, kể cả khi case khác thắng.
2. Nếu có nhiều case sẵn sàng, chọn **một cách giả ngẫu nhiên đều** (uniform pseudo-random) — không phải theo thứ tự viết, không phải round-robin.
3. Nếu không case nào sẵn sàng: có `default` → chạy `default`; không có → block đến khi một case sẵn sàng.
4. `select` với `nil` channel ở mọi case và không có `default` → block mãi.

Hệ quả của (2): **không được dựa vào thứ tự ưu tiên**. Muốn "ưu tiên hủy" thì phải kiểm tra hai lớp:

```go
// Ưu tiên ctx.Done() hơn công việc mới
select {
case <-ctx.Done():
	return ctx.Err()
default:
}
select {
case <-ctx.Done():
	return ctx.Err()
case j := <-jobs:
	process(j)
}
```

**Non-blocking** với `default`:

```go
select {
case ch <- v:
	// sent
default:
	// đầy / không có receiver — bỏ hoặc đếm drop (metric!)
}
```

```go
// try-receive
select {
case v, ok := <-ch:
	if !ok {
		return ErrClosed
	}
	use(v)
default:
	// rỗng
}
```

- `select{}` (không case) — block goroutine hiện tại mãi mãi. Trong `main` thì "giữ process sống"; nếu **mọi** goroutine đều block, runtime báo `fatal error: all goroutines are asleep - deadlock!`.
- Case send trên channel `nil` cũng không bao giờ ready — dùng để tắt nhánh (mục 7).
- `select` chỉ làm việc với channel op. Không "select" được trên mutex, `WaitGroup`, hay syscall.
- Timeout: từ Go 1.23, `time.After` không còn leak timer đến khi fire (`go doc time.After`); hot loop `select` vẫn nên `NewTimer` + `Reset`/`Stop` — xem mục 11.

---

## 7. Nil channel behavior

| Thao tác | Nil channel |
|----------|-------------|
| send `ch <-` | block forever |
| receive `<-ch` | block forever |
| `close(ch)` | panic |
| `len` / `cap` | `0` (không panic) |
| trong `select` | case đó **không bao giờ** ready |

Tắt nhánh động:

```go
var outCh chan int
if needSend {
	outCh = out
}
select {
case outCh <- v: // nếu outCh nil, bỏ qua case này
case <-done:
	return
}
```

Pattern quan trọng nhất của nil channel: **tắt nhánh đã cạn** trong loop `select`. Không có nó, một channel đã đóng sẽ liên tục ready và biến `select` thành busy loop.

```go
// Gộp hai nguồn, thoát khi cả hai đóng — không busy-loop
func merge2(a, b <-chan int, out chan<- int) {
	for a != nil || b != nil {
		select {
		case v, ok := <-a:
			if !ok {
				a = nil // tắt case này vĩnh viễn
				continue
			}
			out <- v
		case v, ok := <-b:
			if !ok {
				b = nil
				continue
			}
			out <- v
		}
	}
	close(out)
}
```

Và **kẹp send phía sau receive** (chỉ bật case send khi thật sự có gì để gửi):

```go
func relay(in <-chan int, out chan<- int, done <-chan struct{}) {
	var pending chan<- int // nil = chưa có gì để gửi
	var v int
	for {
		select {
		case nv, ok := <-in:
			if !ok {
				return
			}
			v, pending = nv, out // bật case send
		case pending <- v:
			pending = nil // tắt lại, quay về nhận
		case <-done:
			return
		}
	}
}
```

Lưu ý: `var ch chan int` là nil, nhưng field channel trong struct cũng nil nếu quên `make` — biểu hiện là "code treo, không panic, không log". Đây là bug hay gặp hơn nhiều so với nil map.

---

## 8. Unidirectional channel types

```go
func produce(out chan<- int) {
	out <- 1
	close(out)
}

func consume(in <-chan int) {
	for v := range in {
		fmt.Println(v)
	}
}

ch := make(chan int)
go produce(ch) // chan → chan<-
consume(ch)    // chan → <-chan
```

- `chan<- T`: chỉ gửi.  
- `<-chan T`: chỉ nhận.  
- Chuyển hai chiều → một chiều được; ngược lại **không** (assignability một chiều). Quy tắc gán chi tiết: [typesystem.md](typesystem.md).
- Giúp API rõ quyền — compiler chặn gửi nhầm từ consumer, và chặn `close` từ phía `<-chan T`.

Đọc kiểu cho đúng: `<-` bám **càng trái càng tốt**, nên `chan<- chan int` là "channel gửi các `chan int`", còn `<-chan <-chan int` là "channel nhận các channel chỉ-nhận". Dùng dấu ngoặc khi mơ hồ: `chan (<-chan int)`.

| Kiểu | Nghĩa |
|------|-------|
| `chan T` | hai chiều |
| `chan<- T` | chỉ gửi (send-only) |
| `<-chan T` | chỉ nhận (receive-only) |
| `chan<- chan T` | gửi được các `chan T` |
| `chan<- (<-chan T)` | gửi được các channel chỉ-nhận |

- Hai kiểu một chiều **không** so sánh/gán chéo nhau, dù cùng gốc.
- `cap`/`len` dùng được trên cả hai kiểu một chiều.
- Return `<-chan T` từ constructor là idiom: caller không đóng được, không gửi được.
- Trong generic code, nếu type parameter được dùng làm channel thì mọi type trong type set phải là channel cho phép thao tác đó, cùng element type (spec). Xem [generics.md](generics.md).

---

## 9. Patterns: fan-in/out, pipeline, done

### Fan-out

Một nguồn, nhiều worker:

```go
jobs := make(chan Job)
for i := 0; i < workers; i++ {
	go func() {
		for j := range jobs {
			process(j)
		}
	}()
}
```

### Fan-in

Nhiều nguồn gộp một channel:

```go
func merge(cs ...<-chan int) <-chan int {
	out := make(chan int)
	var wg sync.WaitGroup
	wg.Add(len(cs))
	for _, c := range cs {
		go func(c <-chan int) {
			defer wg.Done()
			for v := range c {
				out <- v
			}
		}(c)
	}
	go func() {
		wg.Wait()
		close(out)
	}()
	return out
}
```

### Pipeline

```go
func gen(nums ...int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for _, n := range nums {
			out <- n
		}
	}()
	return out
}

func sq(in <-chan int) <-chan int {
	out := make(chan int)
	go func() {
		defer close(out)
		for n := range in {
			out <- n * n
		}
	}()
	return out
}

// usage: for v := range sq(gen(1,2,3)) { ... }
```

### Done channel / cancellation

```go
func worker(done <-chan struct{}, jobs <-chan Job) {
	for {
		select {
		case <-done:
			return
		case j, ok := <-jobs:
			if !ok {
				return
			}
			process(j)
		}
	}
}
```

Hiện đại hơn: dùng `context.Context` — [context.md](context.md).

---

## 10. Khi nào `close`

**Đóng khi:**

- Producer(s) chắc chắn không gửi nữa.  
- Muốn `range` kết thúc.  
- Broadcast “shutdown” qua `close(done)`.

**Không đóng khi:**

- Còn sender khác có thể gửi.  
- Receiver cố đóng (dễ panic double-close / send-on-closed).  
- Channel chỉ dùng nội bộ ngắn và GC sẽ thu — close không bắt buộc để GC, nhưng cần để báo hết dữ liệu.

Quy tắc: **sender closes**. Với nhiều sender → cần coordinator (`WaitGroup` rồi close một lần).

```go
var wg sync.WaitGroup
out := make(chan Result)
for _, t := range tasks {
	wg.Add(1)
	go func(t Task) {
		defer wg.Done()
		out <- do(t)
	}(t)
}
go func() {
	wg.Wait()
	close(out)
}()
```

---

## 11. Pitfalls

1. Deadlock: gửi/nhận unbuffered cùng goroutine không có đối tác.  
2. Send trên closed channel → panic.  
3. Leak: goroutine block trên channel không ai nhận/đóng.  
4. `time.After` trong loop `select` vẫn tạo timer **mỗi lần** vào `select` (chi phí tạo/GC). Từ Go 1.23 timer không còn leak đến khi fire, nhưng hot loop vẫn nên dùng `time.NewTimer` + `Reset`/`Stop`.  
5. Đóng từ nhiều goroutine không đồng bộ.  
6. Buffered đầy + không ai nhận + không select default → block.  
7. Giả định thứ tự giữa nhiều case `select` — không xác định khi cùng ready.

---

## 12. Best practices

1. API nhận `<-chan` / gửi `chan<-` khi có thể.  
2. Document ai đóng channel.  
3. Prefer `struct{}` cho signal.  
4. Kết hợp `context` để hủy.  
5. Tránh channel “thần thánh” — đôi khi Mutex đơn giản hơn.  
6. Buffer size có lý do (đo/backpressure), không số thần kỳ.  
7. Test với `-race` và hủy giữa chừng.

### Cheat sheet

| Cú pháp | Ý nghĩa |
|---------|---------|
| `make(chan T)` | unbuffered |
| `make(chan T, n)` | buffered |
| `ch <- v` / `<-ch` | gửi / nhận |
| `close(ch)` | hết gửi |
| `v, ok := <-ch` | còn dữ liệu? |
| `for range ch` | tới khi đóng |
| `select` | đa kênh / timeout |
| `nil` chan trong select | tắt case |
| `chan<-` / `<-chan` | một chiều |
