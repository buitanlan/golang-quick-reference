# Build constraints trong Go

Build constraints (build tags) quyết định **file nào được đưa vào** lần biên dịch.  
Dùng cho đa nền tảng (`GOOS`/`GOARCH`), tùy chọn tính năng, cgo, sanitizer, và file sinh mã.

Tài liệu này nhắm **Go 1.26** (`go1.26.5`). Language version trong `go.mod`: [typesystem.md](typesystem.md), [packages-modules.md](packages-modules.md). Tham chiếu chính thức: `go help buildconstraint`.

---

## Mục lục

- [Build constraints trong Go](#build-constraints-trong-go)
  - [Mục lục](#mục-lục)
  - [1. Tổng quan](#1-tổng-quan)
  - [2. Cú pháp `//go:build` (hiện đại)](#2-cú-pháp-gobuild-hiện-đại)
  - [3. Legacy `// +build` (obsolete)](#3-legacy--build-obsolete)
  - [4. `GOOS` \& `GOARCH`](#4-goos--goarch)
  - [5. Custom tags \& `-tags`](#5-custom-tags--tags)
  - [6. Hậu tố tên file (`_windows.go`, …)](#6-hậu-tố-tên-file-_windowsgo-)
  - [7. Tag `ignore`](#7-tag-ignore)
  - [8. cgo, `race`, `msan`, `asan`, `unix`](#8-cgo-race-msan-asan-unix)
  - [9. `go1.x` constraints vs `go` directive](#9-go1x-constraints-vs-go-directive)
  - [10. `GOEXPERIMENT` vs `GODEBUG`](#10-goexperiment-vs-godebug)
  - [11. Compiler / tool pragmas](#11-compiler--tool-pragmas)
  - [12. Troubleshooting: file “biến mất”](#12-troubleshooting-file-biến-mất)
  - [13. Kết hợp thực tế \& checklist](#13-kết-hợp-thực-tế--checklist)

---

## 1. Tổng quan

Một package có thể chứa nhiều file; toolchain chọn tập file theo:

1. **Build constraint** ở đầu file (`//go:build ...`).
2. **Filename suffixes** (ví dụ `foo_windows_amd64.go`) — implicit constraint.
3. Flag `-tags` / `-race` / `-msan` / `-asan` và `CGO_ENABLED` khi `go build` / `go test` / `go list`.

```
package fsutil
  path.go              # luôn (nếu không có constraint)
  path_unix.go         # unix-like
  path_windows.go      # Windows
  path_stub.go         # //go:build !windows && !unix
```

- Constraint nằm ở **phần đầu file**, trước `package`, chỉ được đứng sau blank line / comment khác; **phải** có dòng trống trước `package` (tránh lẫn package doc).
- Áp dụng theo **từng file**, không phải theo hàm.
- Chỉ **một** dòng `//go:build` mỗi file (nhiều dòng → lỗi).
- Kiểm tra tập file bằng `go list -f '{{.GoFiles}}'` / `'{{.IgnoredGoFiles}}'` — mục 12.

---

## 2. Cú pháp `//go:build` (hiện đại)

Từ Go 1.17+, dùng `//go:build` (biểu thức boolean):

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

Toán tử:

| Toán tử | Nghĩa |
|---|---|
| `&&` | AND |
| `\|\|` | OR |
| `!` | NOT |
| `()` | nhóm |

Tags được thỏa trong một lần build (`go help buildconstraint`):

- `GOOS`: `windows`, `linux`, `darwin`, `js`, …
- `GOARCH`: `amd64`, `arm64`, `386`, `wasm`, …
- Feature arch: `amd64.v2`, `arm.7`, … (theo `GOAMD64`/`GOARM`/…)
- `unix` — khi `GOOS` là Unix-like
- `gc` hoặc `gccgo`
- `cgo` — khi cgo được hỗ trợ (`CGO_ENABLED`)
- `go1.1`, `go1.22`, `go1.26`, … — theo **toolchain** đang chạy (không có tag riêng cho beta/minor)
- Tags thêm từ `-tags`

```go
//go:build go1.22

package compat
```

Sau dòng `//go:build` **phải** có dòng trống rồi mới tới `package`:

```go
//go:build linux

package host
```

Convention thêm: `purego` — bản thuần Go (không ràng buộc cgo/`unsafe` theo nghĩa tag; dùng khi package có bản assembly).

---

## 3. Legacy `// +build` (obsolete)

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

- `gofmt` vẫn có thể thêm `//go:build` tương đương khi gặp `// +build`.
- Từ Go 1.18+, nếu có cả hai dạng, chúng phải **tương đương**.
- Khuyến nghị: chỉ giữ `//go:build`. Analyzer `plusbuild` của `go fix` (1.26) có thể gỡ comment `// +build` obsolete.

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

> Tag `unix` được toolchain định nghĩa cho nhiều hệ Unix-like (linux, darwin, freebsd, …).

Ánh xạ đặc biệt (`go help buildconstraint`):

- `GOOS=android` cũng thỏa tag/file `linux`
- `GOOS=illumos` cũng thỏa `solaris`
- `GOOS=ios` cũng thỏa `darwin`

```bash
go tool dist list
go env GOOS GOARCH
```

Chứng minh bằng `go list` (Windows host, package demo):

```bash
# mặc định GOOS=windows
go list -f '{{.GoFiles}}' .
# [path.go path_windows.go]

GOOS=linux go list -f '{{.GoFiles}}' .
# [path.go path_linux.go path_unix.go]

GOOS=darwin go list -f '{{.GoFiles}}' .
# [path.go path_unix.go]
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

## 5. Custom tags & `-tags`

Tự định nghĩa tag tính năng:

```go
//go:build integration

package db_test

func TestAgainstRealPostgres(t *testing.T) { /* ... */ }
```

```bash
go test ./...                         # bỏ file integration
go test -tags=integration ./...
go test -tags=integration,debug ./...
```

`-tags` nhận danh sách **phẩy**; dạng cách nhau bằng space đã deprecated nhưng vẫn được nhận (`go help build`).

Feature flag biên dịch:

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

- Đặt tên tag ngắn, hạ chữ, không trùng `GOOS`/`GOARCH`/`unix`/`cgo`/`race`/…
- Document rõ trong README dự án; CI matrix với `-tags=...`.
- Tránh lạm dụng: quá nhiều tổ hợp tag → ma trận build phức tạp.

---

## 6. Hậu tố tên file (`_windows.go`, …)

Ngoài comment, tên file mang constraint ngầm. Sau khi bỏ extension và hậu tố `_test` tùy chọn, nếu khớp:

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
```

```go
// file: sys_windows.go
package sys

func Hostname() (string, error) {
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

- Suffix `GOOS`/`GOARCH` là **ngầm**; không bắt buộc lặp `//go:build windows` trên `_windows.go` (có thể ghi thêm cho rõ).
- File `foo_windows_test.go` = test + windows.
- Tránh tên kiểu `my_linux_helper.go` nếu `linux` không phải ý OS — toolchain diễn giải nhầm suffix. Đặt `_linux.go` đúng chuẩn, hoặc tránh từ khóa OS ở cuối stem.
- Chỉ các giá trị `GOOS`/`GOARCH` **đã biết** của Go mới kích hoạt rule; `foo_bar.go` với `bar` không phải OS/ARCH thì vẫn luôn được xét (trừ constraint khác).

---

## 7. Tag `ignore`

```go
//go:build ignore

package main

// file tiện ích chạy tay / template cho go generate
```

- `ignore` → file **không bao giờ** thuộc package khi `go build` thông thường (mọi từ không thỏa khác cũng được, nhưng `ignore` là convention — `go help buildconstraint`).
- Hay dùng với file generator standalone:

```go
//go:build ignore

package main

import (
	"os"
	"text/template"
)

func main() {
	_ = template.New("x")
	_ = os.Args
}
```

```bash
go run generate.go   # vẫn chạy được khi chỉ định file tường minh
```

Khác `ignore` trong `go.mod` (1.25+): directive module bỏ **thư mục** khỏi pattern `./...` — [packages-modules.md](packages-modules.md).

---

## 8. cgo, `race`, `msan`, `asan`, `unix`

### cgo

File dùng cgo thường import `"C"`:

```go
package zstd

/*
#cgo LDFLAGS: -lzstd
#include <zstd.h>
*/
import "C"
```

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

- Tag `cgo` thỏa khi cgo được hỗ trợ (`CGO_ENABLED`, xem `go help environment`).
- Cross-compile với cgo cần cross C toolchain.
- `go list` tách `CgoFiles` (import `"C"`) khỏi `GoFiles`.

### `race` / `msan` / `asan`

Flag build (`go help build`):

| Flag | Mục đích | Tag thường dùng với `//go:build` |
|---|---|---|
| `-race` | race detector | `race` |
| `-msan` | MemorySanitizer | `msan` |
| `-asan` | AddressSanitizer | `asan` |

```go
//go:build race

package p

func RaceOnly() {}
```

```bash
go list -f '{{.GoFiles}}' .
# race.go nằm trong IgnoredGoFiles

go list -tags=race -f '{{.GoFiles}}' .
# [...] race.go ...]
```

- `-race` / `-msan` / `-asan` có hạn chế OS/ARCH (xem `go help build`); trên Windows, `-race` yêu cầu cgo (`CGO_ENABLED=1`).
- Install suffix tự thêm `_race` / tương tự khi dùng các flag này.

### `unix`

Gom file chung cho Linux/macOS/BSD thay vì OR dài các `GOOS` — mục 4.

---

## 9. `go1.x` constraints vs `go` directive

Hai cơ chế khác nhau:

| Cơ chế | Điều khiển |
|---|---|
| `go` trong `go.mod` / `go.work` | Language version mặc định của module/workspace — [typesystem.md](typesystem.md) |
| `//go:build go1.x` | (1) File chỉ vào build nếu toolchain thỏa tag `go1.x`; (2) với module `go 1.21+`, language version **của file đó** nâng lên tối thiểu theo constraint |

```go
//go:build go1.26

package p

// file này dùng cú pháp/API cần language ≥ 1.26
```

Ví dụ thực tế (`go 1.22` trong `go.mod`, toolchain `go1.26.5`, host Windows):

```bash
go list -f 'GoFiles={{.GoFiles}} Ignored={{.IgnoredGoFiles}}' .
# GoFiles=[base.go newfeat.go] Ignored=[nowin.go]
# newfeat.go có //go:build go1.26 → vẫn được chọn vì toolchain ≥ 1.26
# language version khi compile newfeat.go được nâng theo constraint
```

- Tag `go1.x` **không** bị “tắt” chỉ vì dòng `go` trong `go.mod` thấp hơn — miễn toolchain đang chạy thỏa `go1.x`.
- Modernizer `go fix` (1.26) cũng chỉ đề xuất fix tính năng mới trên file đã yêu cầu đủ version — [packages-modules.md](packages-modules.md) §16.
- Không có tag riêng cho patch (`go1.26.5`); chỉ major.minor dạng `go1.26`.

---

## 10. `GOEXPERIMENT` vs `GODEBUG`

| | `GOEXPERIMENT` | `GODEBUG` |
|---|---|---|
| Khi nào | **Build-time** (toolchain experiments) | **Run-time** (và mặc định compile vào binary) |
| Ổn định | Không — list đổi tùy release; dành cho phát triển toolchain | Cơ chế tương thích chính thức |
| Set bằng | env `GOEXPERIMENT=...` lúc build | env `GODEBUG`, `godebug` trong `go.mod`/`go.work`, `//go:debug` |
| `go env -w` | Không khuyến khích dùng cho production | `GODEBUG` **không** set được bằng `go env -w` |

```bash
# ví dụ experiment (tên cụ thể xem GOROOT/src/internal/goexperiment/flags.go)
GOEXPERIMENT=greenteagc go build .
```

Chi tiết `godebug` / `//go:debug`: [packages-modules.md](packages-modules.md) §13 và `GOROOT/doc/godebug.md`.

---

## 11. Compiler / tool pragmas

Không phải build constraint, nhưng cùng họ comment `//go:...` — hay đi kèm file `ignore` / codegen.

### `//go:generate`

```go
package api

//go:generate stringer -type=Status
//go:generate go run ../tools/gen_openapi.go

type Status int
```

```bash
go generate ./...
```

- Không chạy tự động khi `go build` — phải gọi tường minh.
- Directive phải đầu dòng (chỉ whitespace trước; không space trong `//go`).
- Biến: `$GOFILE`, `$GOPACKAGE`, `$GOLINE`, `$GOOS`, `$GOARCH`, `$GOROOT`, `$DOLLAR`, `$PATH`.
- Pin tool: `go tool stringer` (sau khi khai báo `tool` trong `go.mod`) hoặc `go run example.com/tool@v1.2.3`.

### `//go:embed`

```go
import "embed"

//go:embed hello.txt
var f embed.FS
```

- Cần import `"embed"` (hoặc `_ "embed"` với `string`/`[]byte`).
- Pattern `path.Match`; file nằm trong cây package. Chi tiết: `go doc embed`.

### `//go:noinline`

```go
//go:noinline
func hotspot() { /* ... */ }
```

- Cấm inline call tới func — chủ yếu debug compiler / runtime đặc thù (`go doc cmd/compile`).

### `//go:linkname` (siết từ Go 1.23)

```go
import _ "unsafe"

//go:linkname localname [importpath.name]
```

- Cho phép hai symbol Go alias cùng object-file symbol; phá encapsulation — **chỉ** bật trong file đã import `"unsafe"`.
- Từ **Go 1.23**: linker mặc định `-checklinkname=1` — không cho “pull” symbol nội bộ của **standard library** trừ khi phía định nghĩa đã có `//go:linkname` (hoặc thuộc allowlist tương thích). Tắt kiểm tra (debug): `-ldflags=-checklinkname=0`.
- Không dùng trong application code thông thường.

### `//go:fix inline` (Go 1.26)

```go
// Deprecated: prefer Pow(x, 2).
//go:fix inline
func Square(x int) int { return Pow(x, 2) }
```

- Đánh dấu func/const để analyzer `inline` của `go fix` thay call site bằng body — migration API.
- Áp dụng: `go fix -inline ./...` hoặc `go fix ./...`.
- Xem `go tool fix help inline` và [packages-modules.md](packages-modules.md) §16.

---

## 12. Troubleshooting: file “biến mất”

Khi symbol “thiếu” lúc build nhưng file vẫn nằm trên đĩa — thường do **silent exclusion**. Kiểm tra:

```bash
go list -f '{{.GoFiles}}' ./pkg
go list -f '{{.IgnoredGoFiles}}' ./pkg
go list -f '{{.CgoFiles}}' ./pkg

GOOS=linux go list -f '{{.GoFiles}}' ./pkg
go list -tags=integration -f '{{.GoFiles}}' ./pkg
```

Output thực tế (demo, host `windows/amd64`):

```text
# mặc định
GoFiles: [path.go path_windows.go]
IgnoredGoFiles: [gen.go integ.go path_linux.go path_unix.go race.go]
# gen.go có //go:build ignore; integ.go cần -tags=integration; …

# GOOS=linux
GoFiles: [path.go path_linux.go path_unix.go]

# -tags=integration
GoFiles: [integ.go path.go path_windows.go]
```

Checklist nhanh:

1. Sai `GOOS`/`GOARCH` hoặc thiếu `-tags=...`.
2. Filename suffix OS/ARCH ngoài ý muốn.
3. `//go:build ignore` / `!cgo` với `CGO_ENABLED=0`.
4. Thiếu dòng trống sau `//go:build` → constraint có thể bị parse sai / lẫn doc.
5. Nhầm package (`package foo_test`) hoặc file trong `testdata` / `_` / `.`.
6. Module `ignore` (1.25) loại cả thư mục khỏi `./...` — khác tag file.

---

## 13. Kết hợp thực tế & checklist

Pattern đa nền tảng:

```
pkg/
  file.go           # API chung
  file_stub.go      # //go:build !(windows || unix)
  file_windows.go
  file_unix.go
```

Pattern optional driver:

```go
//go:build sqlite && cgo
```

Checklist:

1. Ưu tiên `//go:build` biểu thức rõ ràng; gỡ `// +build` legacy khi có thể.
2. Đặt dòng trống trước `package`.
3. Dùng filename suffix cho OS/Arch thay vì copy logic runtime nếu có thể.
4. Custom tags: document + CI matrix (`-tags=...`).
5. `ignore` (tag) cho file one-off / generator; `ignore` (go.mod) cho thư mục non-Go trong-tree.
6. Phân biệt `go1.x` (toolchain + optional lang bump) với dòng `go` trong `go.mod`.
7. Đừng nhầm **build-time** exclusion với `if runtime.GOOS`.
8. Xác minh bằng `go list` trước khi đoán — mục 12.

---
