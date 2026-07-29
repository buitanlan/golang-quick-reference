# Package & Module trong Go

Go tổ chức mã nguồn quanh **package** (đơn vị biên dịch/API) và **module** (đơn vị phiên bản & phụ thuộc).  
Hiểu rõ visibility, import path, `go.mod`/`go.work`, và `internal` là nền tảng làm việc với codebase lớn.

Tài liệu này nhắm **Go 1.26** (`go1.26.5`). Directive `go` trong `go.mod` chọn **language version** — xem [typesystem.md](typesystem.md). Build tags / file selection: [build-constraints.md](build-constraints.md).

---

## Mục lục

- [Package \& Module trong Go](#package--module-trong-go)
  - [Mục lục](#mục-lục)
  - [1. Package declaration](#1-package-declaration)
  - [2. Visibility: exported / unexported](#2-visibility-exported--unexported)
  - [3. Import paths \& cú pháp import](#3-import-paths--cú-pháp-import)
    - [3.1 Alias import](#31-alias-import)
    - [3.2 Blank import `_`](#32-blank-import-)
    - [3.3 Dot import `.` (không khuyến khích)](#33-dot-import--không-khuyến-khích)
  - [4. Module: `go.mod`](#4-module-gomod)
  - [5. `go get`, phiên bản \& semantic import versioning](#5-go-get-phiên-bản--semantic-import-versioning)
  - [6. `replace`, `exclude` \& `retract`](#6-replace-exclude--retract)
  - [7. Workspace: `go.work`](#7-workspace-gowork)
  - [8. Internal packages](#8-internal-packages)
  - [9. `init()` \& thứ tự khởi tạo](#9-init--thứ-tự-khởi-tạo)
  - [10. Package documentation \& godoc conventions](#10-package-documentation--godoc-conventions)
  - [11. `tool` directive \& `go tool` (1.24+)](#11-tool-directive--go-tool-124)
  - [12. `ignore` \& package pattern `work` (1.25+)](#12-ignore--package-pattern-work-125)
  - [13. `godebug` \& `//go:debug` (1.23+)](#13-godebug--godebug-123)
  - [14. Module graph, `go.sum` \& lệnh chẩn đoán](#14-module-graph-gosum--lệnh-chẩn-đoán)
  - [15. Proxy, checksum \& module riêng tư](#15-proxy-checksum--module-riêng-tư)
  - [16. `go fix` modernizers (1.26)](#16-go-fix-modernizers-126)
  - [17. Mẹo \& thực hành](#17-mẹo--thực-hành)

---

## 1. Package declaration

Mỗi file `.go` bắt đầu bằng `package <name>`:

```go
package httputil

import "net/http"

func NewClient() *http.Client {
	return &http.Client{}
}
```

- **Tên package** thường là tên thư mục chứa file (không bắt buộc về mặt cú pháp, nhưng là convention mạnh).
- Tất cả file trong cùng một thư mục **phải** cùng tên package (trừ file `_test` có thể dùng `package foo_test` — external test).
- `package main` đánh dấu **executable**; các package khác là library.
- Một package có thể gồm nhiều file; chúng được biên dịch như **một đơn vị**.
- Toolchain bỏ qua thư mục/file bắt đầu bằng `.` hoặc `_`, và thư mục tên `testdata` khi match package pattern (`go help packages`).

> File test: `foo_test.go` với `package foo` → white-box test; `package foo_test` → black-box (chỉ API exported).

---

## 2. Visibility: exported / unexported

Go **không** có `public`/`private` keyword. Visibility theo **chữ cái đầu**:

| Identifier | Phạm vi |
|---|---|
| `Client`, `New`, `HTTPStatus` | **Exported** — thấy được từ package khác |
| `client`, `newClient`, `httpStatus` | **Unexported** — chỉ trong cùng package |

```go
package store

type Cache struct { // exported type
	Items map[string]any // exported field
	mu    sync.Mutex     // unexported field
}

func NewCache() *Cache { // exported constructor
	return &Cache{Items: make(map[string]any)}
}

func (c *Cache) get(key string) (any, bool) { // unexported method
	c.mu.Lock()
	defer c.mu.Unlock()
	v, ok := c.Items[key]
	return v, ok
}
```

- Áp dụng cho: type, func, method, field, const, var.
- Package khác **không** truy cập được identifier unexported, kể cả cùng module.
- Có thể export type nhưng giữ field unexported → encapsulation phổ biến.

---

## 3. Import paths & cú pháp import

Import path là **đường dẫn module + đường dẫn package** (hoặc path chuẩn thư viện):

```go
import (
	"fmt"                         // standard library
	"net/http"

	"example.com/shop/internal/db" // cùng module
	"github.com/jackc/pgx/v5"      // module ngoài
)
```

- Path trong `import` **không** phải đường dẫn filesystem tương đối (`./foo` bị cấm trong module mode).
- Standard library: path ngắn (`fmt`, `encoding/json`).
- Module path lấy từ `module` trong `go.mod`.

### 3.1 Alias import

```go
import (
	jq "encoding/json"
	httptrace "net/http/httptrace"
)

func dump(v any) ([]byte, error) {
	return jq.Marshal(v)
}
```

- Dùng khi: trùng tên package, tên quá chung (`client`), hoặc tránh conflict.
- Alias nên ngắn, có nghĩa; tránh alias “ngẫu nhiên”.

### 3.2 Blank import `_`

Import chỉ để chạy side-effect (`init`), không dùng identifier:

```go
import (
	_ "image/png"              // đăng ký decoder PNG
	_ "github.com/lib/pq"      // đăng ký database/sql driver
	_ "net/http/pprof"         // đăng ký /debug/pprof handlers (khi import đúng chỗ)
)
```

- Compiler **không** cho import package rồi không dùng — trừ `_`.
- Phổ biến với: image codecs, SQL drivers, embed plugins, đăng ký factory.

### 3.3 Dot import `.` (không khuyến khích)

```go
import . "fmt"

func main() {
	Println("hello") // Println thuộc package fmt, đưa vào scope hiện tại
}
```

- **Không khuyến khích** trong production: che giấu nguồn gốc identifier, dễ conflict, khó đọc.
- Chỉ chấp nhận hẹp: test table / DSL rất đặc thù, hoặc generated code có lý do rõ.

---

## 4. Module: `go.mod`

Module là đơn vị versioning. File gốc (các directive phổ biến):

```go
module example.com/shop

go 1.26

toolchain go1.26.5

godebug (
	default=go1.21
	panicnil=1
)

require (
	github.com/jackc/pgx/v5 v5.5.5
	golang.org/x/sync v0.7.0
)

require (
	// indirect dependencies (go mod tidy thêm)
	github.com/jackc/pgpassfile v1.0.0 // indirect
)

tool golang.org/x/tools/cmd/stringer

ignore ./node_modules

exclude github.com/bad/dep v1.0.0

replace github.com/acme/lib => ../acme-lib

retract (
	v1.0.0 // Published by mistake.
)
```

| Directive | Vai trò |
|---|---|
| `module` | Import path prefix của mọi package trong module |
| `go` | **Language version** tối thiểu (+ neo nhiều mặc định tương thích). Từ 1.21 là yêu cầu bắt buộc với toolchain cũ hơn |
| `toolchain` | Gợi ý toolchain cụ thể khi module là main module (1.21+) |
| `require` | Phụ thuộc trực tiếp / gián tiếp |
| `tool` | Khai báo tool chạy bằng `go tool` (1.24+) |
| `ignore` | Bỏ thư mục khỏi package pattern matching (1.25+) |
| `exclude` | Loại một phiên bản module khỏi lựa chọn (chỉ main module) |
| `replace` | Trỏ dependency sang path/local hoặc version khác |
| `retract` | Rút phiên bản đã publish khỏi `@latest` / upgrade tự động |
| `godebug` | Mặc định `GODEBUG` cho work module (1.23+) |

- File kèm: `go.sum` — checksum xác thực nội dung module đã tải (không sửa tay).
- Chỉnh `go.mod` bằng `go get` / `go mod tidy` / `go mod edit`; tránh edit tay trừ khi hiểu rõ.

Lệnh nền:

```bash
go mod init example.com/shop
go mod tidy
go mod download
go mod verify
go list -m all
```

---

## 5. `go get`, phiên bản & semantic import versioning

```bash
go get example.com/lib@v1.2.3
go get example.com/lib@latest
go get example.com/lib@master      # pseudo-version từ commit
go get example.com/lib@none        # gỡ khỏi go.mod (khi không còn import)
go get go@latest                   # nâng dòng go
go get toolchain@patch             # nâng toolchain gợi ý
go get -tool example.com/cmd/foo@v1.2.3   # thêm tool (1.24+)
```

- Go dùng **Minimal Version Selection (MVS)** — chọn phiên bản tối thiểu thỏa mọi ràng buộc.
- **Major v2+**: module path phải có hậu tố `/vN` khớp major (`github.com/foo/bar/v3`). Import path trong code cũng dùng `/vN`. (Major `v0`/`v1` không thêm hậu tố.)
- Pseudo-version dạng: `v0.0.0-yyyymmddhhmmss-commitsha`.
- Từ **Go 1.21+**: dòng `go` = language version; toolchain có thể tự chọn qua `toolchain` + `GOTOOLCHAIN` (mặc định `auto`).

```go
module example.com/shop

go 1.26

toolchain go1.26.5
```

`GOTOOLCHAIN` (xem `go help environment`, [go.dev/doc/toolchain](https://go.dev/doc/toolchain)):

| Giá trị | Ý nghĩa |
|---|---|
| `auto` (mặc định) | = `local+auto`: dùng toolchain bundled; nâng nếu `go`/`toolchain` trong `go.mod`/`go.work` yêu cầu mới hơn |
| `local` | Luôn dùng toolchain đang cài; không tải thêm |
| `path` | = `local+path`: nâng nếu cần nhưng chỉ tìm trên `PATH`, không download |
| `go1.26.5` | Ép đúng toolchain đó |
| `go1.26.5+auto` | Baseline cố định, vẫn cho phép nâng theo `go.mod` |

- Từ **Go 1.25**: khi `go` command cập nhật dòng `go` trong `go.mod`/`go.work`, nó **không còn** tự thêm dòng `toolchain` ghi phiên bản đang chạy.
- Language feature nào được biên dịch phụ thuộc dòng `go` (và có thể `//go:build go1.x` theo file) — chi tiết [typesystem.md](typesystem.md), [build-constraints.md](build-constraints.md).

---

## 6. `replace`, `exclude` & `retract`

```go
module example.com/shop

go 1.26

require github.com/acme/lib v1.4.0

replace github.com/acme/lib => ../acme-lib
replace github.com/acme/lib v1.4.0 => github.com/fork/lib v1.4.1

exclude github.com/bad/dep v1.0.0

retract (
	v1.0.0 // Accidental publish.
	[v1.1.0, v1.1.9] // Broken API.
)
```

- **`replace`**: trỏ dependency sang filesystem local hoặc fork/version khác — hữu ích khi phát triển song song nhiều module.
- **`exclude`**: loại một phiên bản cụ thể khỏi lựa chọn; chỉ có hiệu lực trong **main module** (dependency ignore `exclude` của nhau).
- **`retract`**: author rút version đã publish. Version vẫn tải được nếu đã pin, nhưng `@latest` / upgrade tự động bỏ qua. Nên kèm comment lý do. Publish version **mới hơn** chứa các dòng `retract`.
- `replace` **local path** thường **không** commit lên thư viện public (chỉ cho dev/workspace).
- Trong workspace (`go.work`), ưu tiên `use` + `replace` ở `go.work` thay vì rải `replace` khắp `go.mod`.

Kiểm tra retraction:

```bash
go list -m -u all
go list -m -retracted example.com/m@latest
```

---

## 7. Workspace: `go.work`

Từ Go 1.18, workspace làm việc đồng thời nhiều module:

```go
go 1.26

use (
	./shop
	./shop-api
	../shared/pkg
)

replace example.com/conflict v1.2.3 => example.com/fork v1.2.4

godebug panicnil=1
```

```bash
go work init ./shop ./shop-api
go work use ../shared/pkg
go work sync
go env GOWORK          # path go.work đang dùng, hoặc empty nếu off
```

- Khi có `go.work`, lệnh `go build`/`go test` resolve package theo các module được `use` (các **work modules**).
- Không cần `replace` thủ công giữa các module local trong workspace.
- `replace` / `godebug` trong `go.work` ghi đè / thay thế tương ứng từ từng `go.mod` (với `godebug`: khi có workspace thì chỉ đọc `go.work`, bỏ qua `godebug` trong `go.mod`).
- `go.work` thường **gitignore** (môi trường local); một số team commit để thống nhất monorepo — tùy convention.
- `GOWORK=off` tắt workspace; `GOWORK=/path/to/go.work` chỉ định tường minh.

---

## 8. Internal packages

Package nằm dưới thư mục tên `internal` chỉ được import bởi cây cha của `internal`:

```
example.com/shop/
  go.mod
  cmd/api/main.go
  internal/
    auth/auth.go      # chỉ import được trong cây chứa internal
  pkg/publicapi/
```

```go
// OK trong example.com/shop/cmd/api
import "example.com/shop/internal/auth"

// LỖI nếu module khác cố:
// import "example.com/shop/internal/auth"
```

- Quy tắc: importer phải nằm trong thư mục cha của `internal/` (cùng hoặc dưới tree đó).
- Dùng `internal` để ẩn implementation, giữ API public gọn.
- Có thể có nhiều `internal` ở các cấp khác nhau.

---

## 9. `init()` & thứ tự khởi tạo

```go
package config

import "os"

var Env string

func init() {
	Env = os.Getenv("APP_ENV")
	if Env == "" {
		Env = "dev"
	}
}
```

Thứ tự khởi tạo package (rút gọn):

1. Import dependencies theo thứ tự depth-first, tránh cycle.
2. Với mỗi package: khởi tạo biến package-level theo thứ tự dependency trong file, rồi theo thứ tự file.
3. Chạy tất cả `func init()` của package (có thể nhiều `init` trong một package/file).
4. Cuối cùng: `main.main()` (nếu là program).

- `init` **không** nhận tham số, **không** trả giá trị, **không** gọi tường minh.
- Tránh logic nặng / I/O blocking trong `init` — khó test, khó kiểm soát lỗi.
- Blank import thường dựa vào `init` để đăng ký side-effect.

Cycle import (`A → B → A`) là **lỗi biên dịch**.

---

## 10. Package documentation & godoc conventions

```go
// Package httputil cung cấp helper quanh net/http cho client nội bộ.
//
// Phần lớn hàm là thin wrapper nhằm thống nhất timeout và tracing.
package httputil

// Client bọc http.Client với defaults an toàn hơn.
type Client struct {
	HTTP *http.Client
}

// New tạo Client với timeout 10s.
func New() *Client {
	return &Client{HTTP: &http.Client{Timeout: 10 * time.Second}}
}
```

Conventions:

- Comment package bắt đầu bằng `Package <name> ...` ngay trước `package` clause.
- Doc comment cho exported symbol: câu đầy đủ, bắt đầu bằng tên symbol (`// New tạo...`).
- Đặt `doc.go` chỉ chứa package comment khi package có nhiều file.
- Ví dụ trong doc: dùng thụt đầu dòng để tạo code block trong `go doc` / pkg.go.dev.
- Command `go doc`, `pkgsite` / pkg.go.dev render từ comments này.

```bash
go doc fmt.Println
go doc -all ./internal/auth
```

---

## 11. `tool` directive & `go tool` (1.24+)

Từ **Go 1.24**, khai báo tool trong `go.mod` để version-pin và chạy qua `go tool` (không cần `go install` global):

```go
tool golang.org/x/tools/cmd/stringer

tool (
	example.com/module/cmd/a
	example.com/module/cmd/b
)
```

```bash
go get -tool golang.org/x/tools/cmd/stringer@v0.33.0
go tool stringer -type=Status
go tool                      # liệt kê builtin + tool trong module
go list tool                 # meta-pattern: các tool trong go.mod
```

- Tool ngoài module cần `require` (thường `// indirect`) kèm `tool`.
- Pattern `tool` mở rộng thành danh sách tool trong `go.mod` hiện tại (workspace: union các module trong workspace).
- **Go 1.25+**: distribution kèm ít binary tool prebuilt hơn; nhiều tool được build/chạy qua `go tool` khi cần. Core toolchain (`compile`, `link`, `asm`, `cgo`, `cover`, `vet`, `fix`, …) vẫn có sẵn.

---

## 12. `ignore` & package pattern `work` (1.25+)

### `ignore` trong `go.mod`

Từ **Go 1.25**, bỏ thư mục khỏi việc match package pattern (`./...`, `all`, `work`, …) nhưng file vẫn nằm trong module tree / module zip:

```go
ignore ./node_modules

ignore (
	static
	content/html
	./third_party/javascript
)
```

- Path bắt đầu `./` → relative tới module root.
- Path không có `./` → mọi thư mục cùng tên ở **mọi độ sâu** bị ignore.
- Chỉ có hiệu lực trong **main module**; `ignore` trong dependency bị bỏ qua.
- Khác build tag `//go:build ignore` (loại **file** khỏi package) — xem [build-constraints.md](build-constraints.md).

```bash
go list ./...     # không còn package dưới ./node_modules
go mod edit -ignore=./node_modules
```

### Package pattern `work`

Từ **Go 1.25**, pattern `work` khớp mọi package trong **work module(s)**:

- Module mode: chỉ main module (không kéo dependency như `all`).
- Workspace mode: union các module được `use`.

```bash
go list work          # packages trong work module(s)
go test work          # test các package đó
go list all           # work modules + dependencies (+ test deps)
go list std
go list tool
```

Các reserved pattern khác (`go help packages`): `all`, `std`, `cmd`, `tool`, và tên package `main`.

---

## 13. `godebug` & `//go:debug` (1.23+)

`GODEBUG` điều khiển hành vi tương thích / debug của runtime và một số package (xem `GOROOT/doc/godebug.md`, `go help environment`). Có ba lớp:

1. Biến môi trường `GODEBUG=key=value,...` (runtime, không set được bằng `go env -w`).
2. Directive `godebug` trong `go.mod` / `go.work` (**Go 1.23+**) — mặc định cho work module / workspace.
3. `//go:debug key=value` trong source của **main package** (**Go 1.21+**).

```go
godebug (
	default=go1.21
	panicnil=1
	asynctimerchan=0
)
```

```go
//go:debug default=go1.21
//go:debug panicnil=1

package main
```

- `default=go1.XX` chọn bộ mặc định GODEBUG theo phiên bản, tách khỏi language version ở dòng `go`.
- Chỉ `godebug` của **work module** được đọc; dependency bỏ qua. Có `go.work` → đọc `godebug` từ `go.work`, không từ `go.mod`.
- Key không tồn tại → lỗi. Nhiều `//go:debug` cùng key → lỗi.
- Xem mặc định đã compile vào main:

```bash
go list -f '{{.DefaultGODEBUG}}' ./cmd/app
```

**Không nhầm với `GOEXPERIMENT`**: experiments của toolchain lúc build (không ổn định, dành cho phát triển toolchain) — [build-constraints.md](build-constraints.md).

---

## 14. Module graph, `go.sum` & lệnh chẩn đoán

Từ `go 1.17+`, `go.mod` ghi đủ `require` (kể cả indirect) cho mọi module cung cấp package được import → **module graph pruning** và **lazy module loading**: không cần nạp toàn bộ transitive graph cho mọi lệnh.

`go.sum` chứa hash nội dung module + `go.mod` của dependency. `go mod verify` đối chiếu cache với sum.

```bash
go mod tidy                 # đồng bộ require/sum theo import thực tế
go mod tidy -diff           # in unified diff, không ghi file
go mod tidy -go=1.26        # đồng thời cập nhật dòng go
go mod why github.com/x/y   # đường import ngắn nhất từ main module
go mod why -m github.com/x/y
go mod graph                # cạnh module requirement (sau replace)
go list -m all              # build list
go list -m -u all           # kèm bản mới hơn / retract info
go list -m -json example.com/lib@latest
go mod vendor               # copy dependency vào vendor/
go mod verify
```

- `go mod vendor`: tạo `vendor/`; build với `-mod=vendor` (hoặc khi `vendor/` có mặt tùy cấu hình). Không gồm test của dependency.
- Đừng commit `go.sum` lệch tay; để `tidy`/`get` quản lý.
- Module bị prune vẫn có thể xuất hiện trong graph của dependency cũ (`go 1.16` trở xuống) — nâng `go` giúp graph gọn hơn.

---

## 15. Proxy, checksum & module riêng tư

Biến môi trường (`go help environment`, `go env`):

| Biến | Vai trò |
|---|---|
| `GOPROXY` | Danh sách proxy, cách nhau bởi `,` hoặc `|` (mặc định thường `https://proxy.golang.org,direct`) |
| `GOSUMDB` | Checksum DB (mặc định `sum.golang.org`); `off` để tắt |
| `GOPRIVATE` | Glob path.Match — module private: không qua public proxy/sum theo mặc định |
| `GONOPROXY` | Không dùng proxy (subset tinh chỉnh; mặc định kế thừa `GOPRIVATE`) |
| `GONOSUMDB` | Không kiểm checksum DB |
| `GOINSECURE` | Cho phép fetch insecure (HTTP); **không** tắt sum DB |
| `GOMODCACHE` | Cache module đã tải |

```bash
go env GOPROXY GOSUMDB GOPRIVATE
go env -w GOPRIVATE=github.com/myorg/*,gitlab.internal.example.com/*
```

- Module private: set `GOPRIVATE` (hoặc `GONOPROXY`/`GONOSUMDB`) trước khi `go get`.
- `direct` trong `GOPROXY` = fetch thẳng VCS khi proxy trước đó fail / không phục vụ.

---

## 16. `go fix` modernizers (1.26)

Từ **Go 1.26**, `go fix` được viết lại trên cùng analysis framework với `go vet`: áp dụng **modernizers** (sửa idiom cũ → API/ngôn ngữ mới) thay vì bộ fixer lịch sử đã obsolete.

```bash
go fix ./...
go fix -diff ./...              # chỉ in patch
go fix -newexpr ./...           # một analyzer
go tool fix help                # danh sách analyzer
go tool fix help newexpr
go tool fix help inline
```

Ví dụ analyzer (chạy `go tool fix help` để xem đủ trên toolchain hiện tại): `any`, `minmax`, `rangeint`, `mapsloop`, `slicescontains`, `stringsseq`, `waitgroup`, `newexpr` (`new(expr)` — 1.26), `inline` (theo `//go:fix inline`), `plusbuild`, …

- Modernizer chỉ đề xuất fix trong file đã yêu cầu đủ language version (`go` trong `go.mod` hoặc `//go:build go1.x` trên file) — tránh “nhảy cóc” tính năng.
- `//go:fix inline` đánh dấu func/const để `go fix` (analyzer `inline`) thay call site bằng body — migration API tự phục vụ; chi tiết pragma ở [build-constraints.md](build-constraints.md).
- Không đổi semantics; nếu sai → báo cáo bug.

---

## 17. Mẹo & thực hành

- Một thư mục = một package; tránh “mega package” chứa mọi thứ.
- Tên package ngắn, hạ chữ, không stutter: `http.Server` thay vì `http.HTTPServer` nếu có thể.
- Import group: stdlib → blank line → third-party → blank line → local (tool như `goimports` tự sắp).
- `go mod tidy` trước khi commit; không chỉnh `go.sum` tay.
- Public API ổn định → semantic versioning; breaking change major → `/vN` trong path.
- Dùng `internal` thay vì “hy vọng người khác không import”.
- Hạn chế `init` side-effect khó quan sát; ưu tiên `New`/`Register` tường minh khi thiết kế library.
- Pin tool bằng `tool` + `go get -tool`; chạy `go tool <name>`.
- CI: cân nhắc `GOTOOLCHAIN=local` nếu muốn toolchain deterministic (không auto-download).
- `ignore` cho `node_modules` / artifact sinh ra trong-tree; đừng nhầm với `//go:build ignore`.

---
