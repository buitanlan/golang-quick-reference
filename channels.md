# Channel trong Go

Channel là primitive giao tiếp giữa goroutine: **gửi/nhận giá trị** và đồng bộ. “Don’t communicate by sharing memory; share memory by communicating.”

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
  - [10. Khi nào `close`](#10-khi-nào-close)
  - [11. Pitfalls](#11-pitfalls)
  - [12. Best practices](#12-best-practices)

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
go func() { ch <- 1 }() // cần goroutine — không thì deadlock với nhận cùng thread tuần tự sai cách
v := <-ch
```

Buffered:

```go
ch := make(chan int, 2)
ch <- 1
ch <- 2
// ch <- 3 // block nếu chưa có ai nhận
```

- Buffer 1 thường dùng “signal không mất” / debounce đơn giản.  
- Buffer lớn = queue; vẫn cần chiến lược backpressure.

---

## 3. Send & receive

```go
ch <- v      // send
v := <-ch    // receive
v, ok := <-ch // ok=false nếu đóng và hết dữ liệu
```

- Send/receive trên nil → block mãi.  
- Send trên closed → **panic**.  
- Receive trên closed → zero value + `ok=false` (sau khi drain).

```go
len(ch) // số phần tử đang trong buffer
cap(ch) // capacity
```

---

## 4. `close` & nhận từ channel đóng

```go
close(ch)
```

- Chỉ **người gửi** (hoặc coordinator) đóng — không phải mọi receiver.  
- Đóng hai lần → panic.  
- Đóng nil channel → panic.  
- Sau close: nhận hết buffered rồi `ok=false`.

```go
ch := make(chan int, 2)
ch <- 1
close(ch)
v1, ok1 := <-ch // 1, true
v2, ok2 := <-ch // 0, false
```

Signal “xong” không cần payload:

```go
close(done) // mọi waiter <-done unblock
```

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

---

## 6. `select`

Chờ nhiều thao tác channel; chọn nhánh **sẵn sàng** (ngẫu nhiên nếu nhiều):

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

**Non-blocking** với `default`:

```go
select {
case ch <- v:
	// sent
default:
	// đầy / không có receiver — bỏ hoặc đếm drop
}
```

- `select` không có case → block mãi.  
- Empty `select{}` — sleep forever (giữ process).  
- Timeout: `time.After` mỗi lần select tạo timer — trong hot loop dùng `time.NewTimer`.

---

## 7. Nil channel behavior

| Thao tác | Nil channel |
|----------|-------------|
| send `ch <-` | block forever |
| receive `<-ch` | block forever |
| `close(ch)` | panic |
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
- Chuyển hai chiều → một chiều được; ngược lại không.  
- Giúp API rõ quyền — compiler chặn gửi nhầm từ consumer.

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
4. `time.After` trong loop select → leak timer (dùng Timer.Reset).  
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
