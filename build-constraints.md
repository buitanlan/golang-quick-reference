# Build constraints trong Go

Build constraints (build tags) quyết định **file nào được đưa vào** lần biên dịch.  
Dùng cho đa nền tảng (`GOOS`/`GOARCH`), tùy chọn tính năng, cgo, và file sinh mã.

---

## Mục lục

- [Build constraints trong Go](#build-constraints-trong-go)
  - [Mục lục](#mục-lục)
  - [1. Tổng quan](#1-tổng-quan)
  - [2. Cú pháp `//go:build` (hiện đại)](#2-cú-pháp-gobuild-hiện-đại)
  - [3. Legacy `// +build`](#3-legacy--build)
  - [4. `GOOS` \& `GOARCH`](#4-goos--goarch)
  - [5. Custom tags](#5-custom-tags)
  - [6. Hậu tố tên file (`_windows.go`, …)](#6-hậu-tố-tên-file-_windowsgo-)
  - [7. Tag `ignore`](#7-tag-ignore)
  - [8. cgo \& tags liên quan](#8-cgo--tags-liên-quan)
  - [9. `go generate` (overview)](#9-go-generate-overview)
  - [10. Kết hợp thực tế \& checklist](#10-kết-hợp-thực-tế--checklist)

---

## 1. Tổng quan

Một package có thể chứa nhiều file; toolchain chọn tập file theo:

1. **Build constraint** ở đầu file (`//go:build ...`).
2. **Filename suffixes** (ví dụ `foo_windows_amd64.go`).
3. Flag `-tags` khi `go build` / `go test`.

```
package fsutil
  path.go              # luôn (nếu không có constraint)
  path_unix.go         # unix-like
  path_windows.go      # Windows
  path_stub.go         # //go:build !windows && !unix  (ví dụ)
```

- Constraint nằm ở **phần đầu file**, trước `package`, kèm dòng trống sau đó.
- Áp dụng theo **từng file**, không phải theo hàm.

---

## 2. Cú pháp `//go:build` (hiện đại)

Từ Go 1.17+, khuyến nghị dùng `//go:build`:

```go
//go:build linux && amd64

package cpu

func cacheLine() int { return 64 }
```

```go
//go:build windows || darwin

package notify
```

```go
//go:build !(js || wasm)

package sys
```

Toán tử biểu thức:

| Toán tử | Nghĩa |
|---|---|
| `&&` | AND |
| `\|\|` | OR |
| `!` | NOT |
| `()` | nhóm |

Tags đặc biệt / phổ biến:

- `GOOS` values: `windows`, `linux`, `darwin`, `js`, …
- `GOARCH` values: `amd64`, `arm64`, `386`, `wasm`, …
- `cgo` / `!cgo`
- `go1.21`, `go1.22`, … (phiên bản ngôn ngữ)
- custom: `integration`, `debug`, `pro`, …

```go
//go:build go1.21

package compat
```

Sau dòng `//go:build` **phải** có dòng trống rồi mới tới `package`:

```go
//go:build linux

package host
```

Công cụ `gofmt` / `go fix` có thể đồng bộ với dòng `// +build` legacy.

---

## 3. Legacy `// +build`

Trước Go 1.17:

```go
// +build linux,amd64

package cpu
```

```go
// +build windows darwin

package notify
```

```go
// +build !js,!wasm

package sys
```

Khác biệt cú pháp:

- Dấu phẩy `,` = AND.
- Khoảng trắng = OR.
- `!` phủ định.
- Nhiều dòng `+build` = AND giữa các dòng.

```go
// +build linux darwin
// +build amd64

// tương đương gần: (linux OR darwin) AND amd64
```

Khuyến nghị hiện nay: viết `//go:build`, có thể giữ cả hai trong giai chuyển (cùng nghĩa). Từ Go 1.18+, nếu có cả hai, chúng phải **tương đương**.

```go
//go:build linux && amd64
// +build linux,amd64

package cpu
```

---

## 4. `GOOS` & `GOARCH`

Cross-compile bằng biến môi trường:

```bash
GOOS=windows GOARCH=amd64 go build -o app.exe .
GOOS=linux   GOARCH=arm64 go build -o app.linux .
GOOS=js      GOARCH=wasm  go build -o main.wasm .
```

Trong code:

```go
//go:build windows

package pathsep

const Separator = '\\'
```

```go
//go:build unix

package pathsep

const Separator = '/'
```

> Tag `unix` được toolchain định nghĩa cho nhiều hệ Unix-like (linux, darwin, freebsd, …) — hữu ích gom file chung.

Liệt kê:

```bash
go tool dist list
go env GOOS GOARCH
```

`runtime.GOOS` / `runtime.GOARCH` cho nhánh **runtime** (khác build-time exclusion):

```go
import "runtime"

func open() {
	if runtime.GOOS == "windows" {
		// vẫn nằm trong binary mọi OS
	}
}
```

- Build tag: loại file khỏi binary → binary nhỏ, không cần stub.
- `runtime.GOOS`: một binary đa logic; file luôn compile.

---

## 5. Custom tags

Tự định nghĩa tag tính năng:

```go
//go:build integration

package db_test

func TestAgainstRealPostgres(t *testing.T) { /* ... */ }
```

```bash
go test ./...                         # bỏ file integration
go test -tags=integration ./...       # bật
go test -tags="integration,debug" ./...
```

Ví dụ feature flag biên dịch:

```go
//go:build pro

package license

const Edition = "pro"
```

```go
//go:build !pro

package license

const Edition = "community"
```

- Đặt tên tag ngắn, hạ chữ, không trùng `GOOS`/`GOARCH`.
- Document rõ trong README dự án (user bảo không tạo README ở đây — chỉ nhắc convention).
- Tránh lạm dụng: quá nhiều tổ hợp tag → ma trận build phức tạp.

---

## 6. Hậu tố tên file (`_windows.go`, …)

Ngoài comment, tên file cũng mang constraint:

| Pattern | Nghĩa |
|---|---|
| `*_GOOS.go` | chỉ khi `GOOS` khớp |
| `*_GOARCH.go` | chỉ khi `GOARCH` khớp |
| `*_GOOS_GOARCH.go` | cả hai |
| `*_test.go` | file test (không vào binary thường) |

Ví dụ:

```
conn.go
conn_windows.go
conn_linux.go
conn_darwin_arm64.go
endian_little.go      # nếu dùng tag custom + go:build
```

```go
// file: sys_windows.go
package sys

func Hostname() (string, error) {
	// Windows API
	return windowsHostname()
}
```

```go
// file: sys_unix.go
//go:build unix

package sys

func Hostname() (string, error) {
	return unix.Hostname()
}
```

Lưu ý:

- Suffix `GOOS`/`GOARCH` là **ngầm**; không cần lặp `//go:build windows` nếu đã tên `_windows.go` (có thể vẫn ghi cho rõ).
- File `foo_windows_test.go` = test + windows.
- Tránh tên kiểu `my_linux_helper.go` nếu `linux` không phải ý OS — vì toolchain có thể diễn giải nhầm. Đặt `_linux.go` đúng chuẩn, hoặc tránh từ khóa OS giữa tên.

Các từ bị nhận diện đặc biệt ở cuối tên (trước `.go`): danh sách `GOOS`/`GOARCH` của Go. Ví dụ `foo_bar.go` — `bar` không phải OS thì vẫn OK.

---

## 7. Tag `ignore`

```go
//go:build ignore

package main

// file tiện ích chạy tay / template cho go generate
```

- `ignore` → file **không bao giờ** thuộc package khi `go build` thông thường.
- Hay dùng với file `go generate` standalone:

```go
//go:build ignore

package main

import (
	"os"
	"text/template"
)

func main() {
	// sinh mã...
	_ = template.New("x")
	_ = os.Args
}
```

```bash
go run generare.go   # vẫn chạy được tường minh
```

---

## 8. cgo & tags liên quan

File dùng cgo thường import `"C"`:

```go
package zstd

/*
#cgo LDFLAGS: -lzstd
#include <zstd.h>
*/
import "C"
```

Tags:

```go
//go:build cgo

package zstd
```

```go
//go:build !cgo

package zstd

// pure Go fallback
```

```bash
CGO_ENABLED=0 go build .     # không cgo; file //go:build cgo bị loại
CGO_ENABLED=1 go build .
```

- Cross-compile với cgo khó hơn (cần cross C toolchain).
- Nhiều thư viện cung cấp cả bản `cgo` và `!cgo`.
- Tag `cgo` được set khi `CGO_ENABLED=1` và toolchain hỗ trợ.

---

## 9. `go generate` (overview)

`go generate` quét directive trong source và chạy lệnh — **không** phải build constraint, nhưng thường đi cùng file `ignore` / codegen:

```go
package api

//go:generate stringer -type=Status
//go:generate go run ../tools/gen_openapi.go

type Status int

const (
	StatusOK Status = iota
	StatusFail
)
```

```bash
go generate ./...
go generate ./internal/api
```

Đặc điểm:

- Chạy trên máy dev/CI; cần tool có trên `PATH` (hoặc `go run`).
- Không chạy tự động khi `go build` — phải gọi tường minh (hoặc script CI).
- Comment `//go:generate` phải đầu dòng (chỉ whitespace trước).
- Biến môi trường hữu ích: `$GOFILE`, `$GOPACKAGE`, `$GOARCH`, `$GOOS`, `$GOLINE`.

```go
//go:generate echo generating $GOFILE in package $GOPACKAGE
```

Best practice:

- Commit code đã generate **hoặc** generate trong CI có verify `git diff --exit-code`.
- File generator: `//go:build ignore` để không lẫn vào package.
- Giữ lệnh generate tái lập được (pinned module tool qua `go run example.com/tool@v1.2.3`).

---

## 10. Kết hợp thực tế & checklist

Pattern đa nền tảng:

```
pkg/
  file.go           # API chung
  file_stub.go      //go:build !(windows || unix)
  file_windows.go
  file_unix.go
```

Pattern optional driver:

```
//go:build sqlite && cgo
```

Checklist:

1. Ưu tiên `//go:build` biểu thức rõ ràng.
2. Đặt dòng trống trước `package`.
3. Dùng filename suffix cho OS/Arch thay vì copy logic runtime nếu có thể.
4. Custom tags: document + CI matrix (`-tags=...`).
5. `ignore` cho file one-off / generator.
6. Kiểm tra bằng:

```bash
go list -f '{{.GoFiles}}' ./pkg
GOOS=windows go list -f '{{.GoFiles}}' ./pkg
go test -tags=integration ./...
```

7. Đừng nhầm **build-time** exclusion với `if runtime.GOOS` — chọn đúng công cụ.

---
