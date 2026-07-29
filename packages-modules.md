# Package & Module trong Go

Go tổ chức mã nguồn quanh **package** (đơn vị biên dịch/API) và **module** (đơn vị phiên bản & phụ thuộc).  
Hiểu rõ visibility, import path, `go.mod`/`go.work`, và `internal` là nền tảng làm việc với codebase lớn.

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
  - [6. `replace` \& `exclude`](#6-replace--exclude)
  - [7. Workspace: `go.work`](#7-workspace-gowork)
  - [8. Internal packages](#8-internal-packages)
  - [9. `init()` \& thứ tự khởi tạo](#9-init--thứ-tự-khởi-tạo)
  - [10. Package documentation \& godoc conventions](#10-package-documentation--godoc-conventions)
  - [11. Mẹo \& thực hành](#11-mẹo--thực-hành)

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

Module là đơn vị versioning. File gốc:

```go
module example.com/shop

go 1.22

require (
	github.com/jackc/pgx/v5 v5.5.5
	golang.org/x/sync v0.7.0
)

require (
	// indirect dependencies (go mod tidy thêm)
	github.com/jackc/pgpassfile v1.0.0 // indirect
)
```

- `module`: import path prefix của mọi package trong module.
- `go`: phiên bản ngôn ngữ tối thiểu (ảnh hưởng ngôn ngữ + một số mặc định toolchain).
- `require`: phụ thuộc trực tiếp / gián tiếp.
- File kèm: `go.sum` — checksum xác thực nội dung module đã tải.

Lệnh thường dùng:

```bash
go mod init example.com/shop
go mod tidy          # đồng bộ require/sum theo import thực tế
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
```

- Go dùng **Minimal Version Selection (MVS)** — chọn phiên bản tối thiểu thỏa mọi ràng buộc.
- **Major v2+**: import path thường kèm `/vN` (`github.com/foo/bar/v3`), trừ module path đặc biệt.
- Pseudo-version dạng: `v0.0.0-yyyymmddhhmmss-commitsha`.
- Từ Go 1.21+: toolchain có thể tự chọn qua `toolchain` directive trong `go.mod`.

```go
module example.com/shop

go 1.22

toolchain go1.22.5
```

---

## 6. `replace` & `exclude`

```go
module example.com/shop

go 1.22

require github.com/acme/lib v1.4.0

replace github.com/acme/lib => ../acme-lib
replace github.com/acme/lib v1.4.0 => github.com/fork/lib v1.4.1

exclude github.com/bad/dep v1.0.0
```

- **`replace`**: trỏ dependency sang filesystem local hoặc fork/version khác — rất hữu ích khi phát triển song song nhiều module.
- **`exclude`**: loại một phiên bản cụ thể khỏi lựa chọn (hiếm dùng hơn `replace`).
- `replace` **local path** thường **không** commit lên thư viện public (chỉ cho dev/workspace).
- Trong workspace (`go.work`), ưu tiên dùng `use` thay vì rải `replace` khắp nơi.

---

## 7. Workspace: `go.work`

Từ Go 1.18, workspace cho phép làm việc đồng thời nhiều module:

```go
go 1.22

use (
	./shop
	./shop-api
	../shared/pkg
)
```

```bash
go work init ./shop ./shop-api
go work use ../shared/pkg
go work sync
```

- Khi có `go.work`, lệnh `go build`/`go test` trong workspace resolve package theo các module được `use`.
- Không cần `replace` thủ công giữa các module local trong workspace.
- `go.work` thường **gitignore** (môi trường local); một số team commit để thống nhất monorepo — tùy convention.

---

## 8. Internal packages

Package nằm dưới thư mục tên `internal` chỉ được import bởi cây cha của `internal`:

```
example.com/shop/
  go.mod
  cmd/api/main.go
  internal/
    auth/auth.go      # chỉ import được trong module shop (cây chứa internal)
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

## 11. Mẹo & thực hành

- Một thư mục = một package; tránh “mega package” chứa mọi thứ.
- Tên package ngắn, hạ chữ, không stutter: `http.Server` thay vì `http.HTTPServer` nếu có thể.
- Import group: stdlib → blank line → third-party → blank line → local (tool như `goimports` tự sắp).
- `go mod tidy` trước khi commit; không chỉnh `go.sum` tay.
- Public API ổn định → semantic versioning; breaking change major → `/vN` trong path.
- Dùng `internal` thay vì “hy vọng người khác không import”.
- Hạn chế `init` side-effect khó quan sát; ưu tiên `New`/`Register` tường minh khi thiết kế library.

---
