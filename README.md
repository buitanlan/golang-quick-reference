# Tài liệu tham khảo ngôn ngữ lập trình Go

Bộ tài liệu tham chiếu **in-depth / advanced** cho ngôn ngữ Go, nhắm **Go 1.27** ([release notes](https://go.dev/doc/go1.27)). Không phải giáo trình nhập môn: các khái niệm được trình bày dạng tham khảo nhanh kèm chi tiết nâng cao (semantics, version gates, pitfalls). Nếu chưa biết Go, bắt đầu bằng Tour of Go / Effective Go bên dưới, rồi dùng bộ này khi cần tra cứu sâu hơn.

Luật kiểu/câu lệnh nào được bật phụ thuộc dòng `go` trong `go.mod` (language version), không phải toolchain đang cài. Trên toolchain **1.26.x**, compiler vẫn từ chối generic method: `method must have no type parameters` — cần `-lang` ≥ `go1.27`.

Bộ này là **tham chiếu ngôn ngữ**. Stdlib 1.27 ngoài scope từng file (nhưng đáng biết khi nâng version): `encoding/json` chuyển sang engine v2 (opt-out `GOEXPERIMENT=nojsonv2`), package `uuid`, `crypto/mldsa`, `encoding/json/v2` + `jsontext`. Chi tiết: [Go 1.27 release notes](https://go.dev/doc/go1.27).

---

Tham khảo chính thức: [A Tour of Go](https://go.dev/tour/) · [Effective Go](https://go.dev/doc/effective_go) · [Release notes](https://go.dev/doc/devel/release) · [Go 1.27](https://go.dev/doc/go1.27)

---

## Nội dung  

- [Package & Module](packages-modules.md)
- [Hàm main](main-function.md)
- [Hệ thống kiểu dữ liệu](typesystem.md)
- [Build constraints](build-constraints.md)
- [Literal & Constant](literals.md)
- [Toán tử](operators.md)
- [Từ khóa](keywords.md)
- [Phát biểu](statements.md)
- [Hàm](functions.md)
- [Method & Interface](methods-interfaces.md)
- [Struct & Composition](structs-composition.md)
- [Pointer](pointers.md)
- [Error handling](errors.md)
- [Tập hợp (array, slice, map)](collections.md)
- [Generics](generics.md)
- [Concurrency (goroutine & sync)](concurrency.md)
- [Channel](channels.md)
- [Context](context.md)
