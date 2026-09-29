# Decisions

Ghi chú các quyết định riêng của fork `mod_iced` (không thuộc upstream).

## 0001 — Cache `wgpu::Instance` + `Adapter` trong `iced_wgpu`

- Ngày: 2026-09-29
- Trạng thái: Đã chốt

### Vấn đề

Mỗi lần tạo `Compositor` (tức mỗi window / mỗi `run()` khởi tạo compositor), `iced_wgpu`
lại tạo mới `wgpu::Instance` và gọi `request_adapter` — tức là dò/tìm GPU driver lại.

### Các phương án đã cân nhắc

- **(A) Giữ `Compositor` sống suốt daemon** (bỏ drop khi hết window).
  Đã thử, **đã revert**: thừa (vẫn tạo lại device/pipeline khi hết window) và giữ GPU
  memory khi idle.
- **(B) Pre-create compositor lúc khởi động daemon.** Loại: `Compositor::request` cần
  `compatible_surface` để chọn format; tạo sẵn không có surface sẽ lỗi
  `IncompatibleSurface`.
- **(C) API inject GPU context từ ngoài.** Loại: phải đục qua top-level `iced`, xâm lấn
  nhiều.
- **(2) Cache `Instance` + `Adapter` process-wide trong `iced_wgpu`.**
  Chốt phương án này.

### Quyết định

Cache trong `iced_wgpu` ở phạm vi process:

- `Instance`: `static OnceLock<Mutex<HashMap<Backends, wgpu::Instance>>>`.
- `Adapter`: `static OnceLock<Mutex<HashMap<AdapterKey, wgpu::Adapter>>>`, với
  `AdapterKey { backends, power_preference }`.
- Cache hit → **bỏ qua** `request_adapter` và `enumerate_adapters`.
- Nếu surface mới không tương thích adapter đã cache (danh sách `formats` rỗng) → fallback
  request lại và cập nhật cache.
- Giữ nguyên `request_device`, chọn format, alpha mode.
- Chỉ cache trên native (`#[cfg(not(target_arch = "wasm32"))]`); wasm giữ nguyên code cũ.

### Chi tiết triển khai

- `wgpu/src/window/compositor.rs`:
  - thêm `AdapterKey`, `new_instance`, `instances()`, `adapters()`.
  - sửa `Compositor::request` để lấy instance/adapter từ cache.
  - lưu ý: giữ `MutexGuard` không được kéo dài qua `match` scrutinee/`.await` để tránh
    deadlock (đã tách `let cached = ...;` trước `match`).
- `wgpu/src/lib.rs`: thêm `#![recursion_limit = "256"]` để dẹp warning
  future-incompat do type wgpu lồng sâu trong static.
- `CHANGELOG.md`: mục `[Unreleased] > Changed`.

### Kiểm chứng

- `cargo check -p iced_wgpu` — pass
- `cargo check -p iced` — pass
- `cargo clippy -p iced_wgpu` — sạch
- Chạy `examples/counter` — exit `0`

### Git

- Commit: `1f9560b4` — branch `wgpu-adapter-cache`
- Tag `0.14.0` → `1f9560b4`
- Đã push fork `xtilam/mod_iced`; xoá branch cũ `fork/compositor-lifetime`.

### Hạn chế còn lại (chưa làm)

- `Device` / `Queue` vẫn tạo mỗi `Compositor` (chưa cache).
- Chưa xử lý adapter/device lost riêng; chỉ fallback khi surface không tương thích.
- Chưa test runtime trên wasm (chỉ đảm bảo build).
