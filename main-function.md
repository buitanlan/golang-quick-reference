# Hàm main trong Go

Chương trình Go executable bắt đầu từ `package main` và hàm `func main()`.
Khác C#/Java, Go **không** nhận `args` qua tham số `main` — dùng `os.Args` hoặc package `flag`.

---

## Mục lục

- [Hàm main trong Go](#hàm-main-trong-go)
  - [Mục lục](#mục-lục)
  - [1. `package main` \& `func main()`](#1-package-main--func-main)
  - [2. Điểm vào chương trình (program entry)](#2-điểm-vào-chương-trình-program-entry)
  - [3. Tham số dòng lệnh: `os.Args`](#3-tham-số-dòng-lệnh-osargs)
  - [4. Package `flag` (overview)](#4-package-flag-overview)
  - [5. Mã thoát: `os.Exit` \& convention](#5-mã-thoát-osexit--convention)
  - [6. Thứ tự `init` trước `main`](#6-thứ-tự-init-trước-main)
  - [7. Nhiều `main` packages (nhiều thư mục)](#7-nhiều-main-packages-nhiều-thư-mục)
  - [8. Build \& chạy](#8-build--chạy)
  - [9. `GOMAXPROCS` \& container (Go 1.25+)](#9-gomaxprocs--container-go-125)
  - [10. Signal \& graceful shutdown](#10-signal--graceful-shutdown)
  - [11. `TestMain`](#11-testmain)
  - [12. Best practices](#12-best-practices)

---

## 1. `package main` & `func main()`

```go
package main

import "fmt"

func main() {
	fmt.Println("Hello, Go")
}
```

Yêu cầu:

- File thuộc `package main`.
- Có đúng một hàm `func main()` **không tham số, không giá trị trả về** trong package đó.
- Chữ ký khác (`func main(args []string)`, `func main() int`) → **không** phải entry point hợp lệ (hoặc lỗi biên dịch nếu cố trả về).

```go
// SAI — không phải entry của Go
func main() int {
	return 0
}
```

- `main` có thể gọi hàm khác, spawn goroutine, lắng nghe signal, v.v.
- Khi `main` return bình thường → process thoát với code **0** (trừ khi đã `os.Exit` trước đó).
- Goroutine còn lại khi `main` return **không** được join — process kết thúc ngay.

---

## 2. Điểm vào chương trình (program entry)

Luồng khởi động (rút gọn):

1. Runtime khởi tạo scheduler, GC, …
2. Khởi tạo các package được import (đệ quy), chạy `init` theo thứ tự.
3. Khởi tạo package `main`, chạy `init` của `main` (nếu có).
4. Gọi `main.main()`.
5. Khi `main` return: chương trình kết thúc; goroutine còn lại bị dừng (không “join” tự động).

```go
package main

import "fmt"

func init() {
	fmt.Println("init main")
}

func main() {
	fmt.Println("main")
}
```

> **Lưu ý**: `defer` trong `main` **có** chạy khi `main` return bình thường. `os.Exit` thoát ngay — **không** chạy `defer` nào còn pending (kể cả trong `main`).

```go
func main() {
	defer fmt.Println("cleanup") // chạy khi return
	fmt.Println("work")
}
```

---

## 3. Tham số dòng lệnh: `os.Args`

```go
package main

import (
	"fmt"
	"os"
)

func main() {
	fmt.Println("program:", os.Args[0])
	fmt.Println("argc:", len(os.Args))
	for i, a := range os.Args[1:] {
		fmt.Printf("arg[%d]=%q\n", i+1, a)
	}
}
```

- `os.Args[0]`: tên chương trình (như được invoke — có thể là path).
- `os.Args[1:]`: các đối số người dùng truyền.
- Kiểu: `[]string` — **không** tách option/value; bạn tự parse hoặc dùng `flag`.

```bash
go run . foo bar
# Args[1]=foo Args[2]=bar
```

---

## 4. Package `flag` (overview)

`flag` là parser flag kiểu Unix trong stdlib:

```go
package main

import (
	"flag"
	"fmt"
	"os"
)

func main() {
	var (
		host    = flag.String("host", "127.0.0.1", "listen host")
		port    = flag.Int("port", 8080, "listen port")
		verbose = flag.Bool("v", false, "verbose logging")
	)
	flag.Parse()

	fmt.Printf("addr=%s:%d verbose=%v\n", *host, *port, *verbose)
	fmt.Println("positional:", flag.Args()) // sau các flag
	_ = os.Args
}
```

Điểm chính:

- Khai báo bằng `flag.String` / `Int` / `Bool` / `Duration` / `Var`…
- Gọi `flag.Parse()` trước khi đọc giá trị.
- Flag dạng `-name` hoặc `--name` (Go chấp nhận cả hai cho flag package).
- `flag.Args()`: tham số positional còn lại.
- `flag.Usage`, `flag.CommandLine` cho tùy biến.

```go
type Config struct {
	Workers int
}

func (c *Config) Set(s string) error {
	_, err := fmt.Sscanf(s, "%d", &c.Workers)
	return err
}
func (c *Config) String() string { return fmt.Sprintf("%d", c.Workers) }
```

Với CLI phức tạp (subcommand), cân nhắc thư viện ngoài (`cobra`, `urfave/cli`, …) — ngoài phạm vi stdlib nhưng phổ biến trong thực tế.

```bash
go run . -host=0.0.0.0 -port=9000 -v serve
```

---

## 5. Mã thoát: `os.Exit` & convention

```go
package main

import (
	"fmt"
	"os"
)

func main() {
	if len(os.Args) < 2 {
		fmt.Fprintln(os.Stderr, "usage: tool <file>")
		os.Exit(2) // usage / misuse — convention phổ biến
	}
	if err := run(os.Args[1]); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
}

func run(path string) error {
	// ...
	return nil
}
```

| Code | Ý nghĩa thông dụng |
|---|---|
| `0` | Thành công (`main` return cũng vậy) |
| `1` | Lỗi chung |
| `2` | Sai cách dùng / argument (nhiều tool Unix) |

**Quan trọng:**

- `os.Exit(code)` **thoát ngay**: **không** chạy `defer`.
- Pattern hay dùng: để `main` mỏng, logic trong `run() error`, rồi `os.Exit` một chỗ — hoặc `return` và map lỗi ở cuối `main` **sau** các `defer`.

```go
func main() {
	os.Exit(realMain())
}

func realMain() int {
	defer cleanup() // vẫn chạy trước khi Exit ở main
	if err := work(); err != nil {
		fmt.Fprintln(os.Stderr, err)
		return 1
	}
	return 0
}
```

---

## 6. Thứ tự `init` trước `main`

```go
// file a.go
package main

import "fmt"

var A = f("A")

func init() { fmt.Println("init A") }

func f(s string) int {
	fmt.Println("var", s)
	return 1
}
```

```go
// file b.go
package main

import "fmt"

func init() { fmt.Println("init B") }

func main() { fmt.Println("main") }
```

Quy tắc (tóm tắt):

1. Tất cả package phụ thuộc khởi tạo xong trước package hiện tại.
2. Trong một package: biến package-level được khởi tạo theo thứ tự phụ thuộc; nhiều `init` chạy theo thứ tự xuất hiện (file được sắp bởi compiler theo tên file).
3. `main.main()` chỉ chạy sau mọi `init` của `main` và dependencies.

Ứng dụng thực tế:

- Đăng ký driver / codec qua blank import + `init`.
- Tránh đọc config mạng trong `init` — lỗi khó báo, khó test.
- Không phụ thuộc thứ tự `init` giữa các file nếu có thể; gom rõ ràng vào một chỗ.

Chi tiết hơn: [packages-modules.md §9](packages-modules.md#9-init--thứ-tự-khởi-tạo).

---

## 7. Nhiều `main` packages (nhiều thư mục)

Một module có thể có **nhiều** chương trình — mỗi cái một thư mục `package main`:

```
example.com/shop/
  go.mod
  cmd/
    api/main.go       # package main → binary api
    worker/main.go    # package main → binary worker
    migrate/main.go
  internal/...
  pkg/...
```

```go
// cmd/api/main.go
package main

func main() { /* HTTP server */ }
```

```go
// cmd/worker/main.go
package main

func main() { /* background jobs */ }
```

- **Không** được đặt hai `package main` (hai `func main`) trong **cùng một thư mục**.
- Convention phổ biến: `cmd/<appname>/`.
- Build từng binary:

```bash
go build -o bin/api ./cmd/api
go build -o bin/worker ./cmd/worker
go install ./cmd/...
```

- Library code để ngoài `cmd` (`internal/`, `pkg/`) để tái sử dụng giữa các binary.

---

## 8. Build & chạy

```bash
go run .                    # biên dịch tạm + chạy package main ở thư mục hiện tại
go run ./cmd/api
go build -o app ./cmd/api
go install ./cmd/api        # vào $GOBIN / $GOPATH/bin
```

Biến môi trường hữu ích:

- `CGO_ENABLED=0` — build không cgo (dễ binary portable hơn).
- `GOOS`/`GOARCH` — cross-compile:

```bash
GOOS=linux GOARCH=amd64 go build -o api.linux ./cmd/api
```

### `-ldflags -X` — embed version

`go tool link` hỗ trợ `-X importpath.name=value` để gán biến `string` đã khai báo (uninitialized hoặc const string):

```bash
go build -ldflags="-X main.version=1.2.3" ./cmd/api
```

```go
package main

var version = "dev"

func main() {
	println(version)
}
```

- Chỉ hiệu lực với biến `string` ở package-level đúng `importpath.name`.
- Không hoạt động nếu initializer gọi hàm hoặc tham chiếu biến khác.

### Build constraints

File chỉ build trên một OS/arch/tag dùng `//go:build` — xem [build-constraints.md](build-constraints.md). Language version của file cũng có thể gắn `//go:build go1.22` (độc lập toolchain).

### `tool` directive (Go 1.24+)

`go.mod` có thể khai báo tool (linter, generator) tách khỏi dependency runtime:

```bash
go get -tool golang.org/x/tools/cmd/stringer@latest
go tool stringer -type=Day
```

- `go get -tool path@version` thêm dòng `tool` vào `go.mod`.
- Chạy bằng `go tool <name> …` — version pin theo module, không phụ thuộc `$PATH` máy dev.

### `go fix` — modernizers (Go 1.26)

Từ 1.26, `go fix` được viết lại trên analysis framework (cùng nền `go vet`) và là nơi chứa **modernizers**:

```bash
go fix ./...
go fix -diff ./...          # chỉ xem patch
go tool fix help            # danh sách analyzer
go tool fix help newexpr    # chi tiết một fixer
```

Ví dụ modernizer liên quan entry/codebase: `newexpr`, `rangeint`, `forvar`, `minmax`, `waitgroup`, `any`, `testingcontext`… Không đổi hành vi có chủ đích — nếu sai, báo issue. Xem thêm [keywords.md](keywords.md).

---

## 9. `GOMAXPROCS` & container (Go 1.25+)

`GOMAXPROCS` = số P (logical processor) — số goroutine chạy CPU **đồng thời**. Default từ Go **1.25** (khi language version ≥ 1.25):

- Nếu env `GOMAXPROCS` là số nguyên dương → dùng giá trị đó (và **tắt** auto-update).
- Không thì runtime chọn min hợp lý từ: số logical CPU, CPU affinity, và trên Linux **cgroup CPU quota** (CPU limit).
- Runtime **định kỳ cập nhật** default khi CPU/affinity/cgroup đổi (tới ~1 lần/giây). Gọi `runtime.GOMAXPROCS(n)` cũng tắt auto-update.
- `runtime.SetDefaultGOMAXPROCS()` (1.25+) khôi phục default + auto-update.
- `GODEBUG=containermaxprocs=0` / `updatemaxprocs=0` giữ hành vi cũ; đó là default cho language version ≤ 1.24.

```go
runtime.GOMAXPROCS(0) // đọc giá trị hiện tại, không đổi
```

Ý nghĩa thực tế: container limit 2 CPU trên host 64 core → process Go 1.25+ thường chạy với ~2 P thay vì 64 — giảm throttling. Chi tiết scheduler: [concurrency.md §13](concurrency.md#13-scheduler-gomaxprocs--metrics).

---

## 10. Signal & graceful shutdown

Pattern chuẩn từ `main`:

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"os"
	"os/signal"
)

func main() {
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt)
	defer stop()

	if err := run(ctx); err != nil && !errors.Is(err, context.Canceled) {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
}

func run(ctx context.Context) error {
	<-ctx.Done()
	// Go 1.26+: Cause mô tả signal, ví dụ "interrupt signal received"
	fmt.Fprintln(os.Stderr, "shutdown:", context.Cause(ctx))
	return ctx.Err()
}
```

- `NotifyContext` (1.16+): Done khi nhận signal, khi `stop()`, hoặc parent hủy — cái nào trước.
- Gọi `stop()` sớm để **unregister** và trả lại default signal behavior (ví dụ Interrupt → exit).
- Go **1.26**: hủy do signal dùng `CancelCauseFunc`; `context.Cause(ctx)` là lỗi mô tả signal. `errors.Is(cause, context.Canceled) == true`. Gọi `stop()` → cause thường `Canceled`.
- Trên Unix thường thêm `syscall.SIGTERM`. Windows: `os.Interrupt` (Ctrl+C) là phổ biến.
- Kết hợp `http.Server.Shutdown(ctx)` / đóng listener — xem [context.md](context.md).

---

## 11. `TestMain`

Package test có thể định nghĩa entry riêng:

```go
func TestMain(m *testing.M) {
	// setup — flag.Parse() nếu TestMain tự đọc flag
	code := m.Run()
	// teardown
	os.Exit(code)
}
```

Theo `go doc testing`:

- Nếu có `TestMain`, test binary gọi nó thay vì chạy test trực tiếp.
- `m.Run()` chạy tests/benchmarks; trả về exit code — thường `os.Exit(code)`. Nếu `TestMain` return mà không `Exit`, wrapper vẫn `os.Exit` với kết quả `m.Run` (chỉ khi đã gọi `Run`).
- Khi `TestMain` bắt đầu, **`flag.Parse` chưa chạy** — gọi tường minh nếu cần flag của `testing` hoặc flag riêng.
- Dùng cho setup global (testcontainer, synctest shard, seed DB) — không cần cho test thường.

`testing.M` / `m.Run` là primitive thấp; phần lớn suite không cần `TestMain`.

---

## 12. Best practices

- Giữ `main` mỏng: parse flag → wire dependencies → `run(ctx)` → map lỗi/exit.
- Dùng `signal.NotifyContext` + `defer stop()`; từ 1.26 đọc `context.Cause` nếu cần biết signal.
- Log lỗi ra `stderr`; output dữ liệu ra `stdout` (friendly với pipe).
- Tránh `panic` cho lỗi thường gặp; `panic` chỉ cho invariant vỡ thật sự.
- Nhiều binary → cấu trúc `cmd/`; chia sẻ code qua `internal`.
- Nhớ: `os.Exit` bỏ qua `defer` — thiết kế đường thoát có chủ đích (`realMain() int`).
- Pin tool bằng `tool` trong `go.mod` (1.24+); hiện đại hóa codebase bằng `go fix` (1.26).
- Container: language version ≥ 1.25 để hưởng default `GOMAXPROCS` theo cgroup.

```go
package main

import (
	"context"
	"fmt"
	"os"
	"os/signal"
)

func main() {
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt)
	defer stop()

	if err := run(ctx); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
}

func run(ctx context.Context) error {
	<-ctx.Done()
	return nil
}
```

### Version map

| Phiên bản | Liên quan `main` / process |
|---|---|
| 1.16 | `signal.NotifyContext` |
| 1.24 | `tool` directive trong `go.mod`; `go get -tool` |
| 1.25 | default `GOMAXPROCS` theo cgroup + auto-update; `SetDefaultGOMAXPROCS` |
| 1.26 | `NotifyContext` gắn Cause theo signal; `go fix` = modernizers (`newexpr`, …) |

### Checklist

```text
□ package main + func main() không tham số/return
□ main mỏng; defer cleanup trước os.Exit (hoặc realMain pattern)
□ NotifyContext + defer stop(); Cause (1.26) nếu cần phân biệt signal
□ -ldflags -X chỉ gắn string package-level hợp lệ
□ //go:build xem build-constraints.md; tool pin trong go.mod khi cần
□ go.mod go 1.25+ nếu muốn GOMAXPROCS theo container
```
