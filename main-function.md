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
  - [9. Best practices](#9-best-practices)

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

> **Lưu ý**: return từ `main` **không** chạy `defer` đã đăng ký *sau* khi `os.Exit` được gọi — vì `os.Exit` thoát ngay. Nhưng `defer` trong `main` **có** chạy khi `main` return bình thường.

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
		host = flag.String("host", "127.0.0.1", "listen host")
		port = flag.Int("port", 8080, "listen port")
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

- `CGO_ENABLED=0` — static-ish build không cgo.
- `GOOS`/`GOARCH` — cross-compile:

```bash
GOOS=linux GOARCH=amd64 go build -o api.linux ./cmd/api
```

Version embedding (thực tế phổ biến):

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

---

## 9. Best practices

- Giữ `main` mỏng: parse flag → wire dependencies → `run(ctx)` → xử lý lỗi/exit.
- Dùng `context.Context` từ `main` (signal: `signal.NotifyContext`) để shutdown sạch.
- Log lỗi ra `stderr`; output dữ liệu ra `stdout` (friendly với pipe).
- Tránh `panic` cho lỗi thường gặp; `panic` chỉ cho invariant vỡ thật sự.
- Nhiều binary → cấu trúc `cmd/`; chia sẻ code qua `internal`.
- Nhớ: `os.Exit` bỏ qua `defer` — thiết kế đường thoát có chủ đích.

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

---
