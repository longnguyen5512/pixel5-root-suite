# Pixel 5 (`redfin`) Android 17 Root & Hardening Suite

Bộ tổng hợp 4 repository phục vụ cài đặt, tối ưu và vượt qua các bài kiểm tra môi trường/attestation trên **Google Pixel 5 (`redfin`) — Android 17 (API 37)** với **KernelSU-Next + SUSFS**.

---

## Danh sách Repository thành phần

| Thành phần | Repository | Nhánh chính | Vai trò |
|---|---|---|---|
| **Kernel** | [`longnguyen5512/306kernels-redbull`](https://github.com/longnguyen5512/306kernels-redbull) | `3.4.2-dev` | Kernel `4.19.325` + KernelSU-Next + SUSFS; vá timing `selinux_hide` (tra cứu 2-slot double-hash không nhánh), tách `get_anon_bdev()` cho mount namespace ngoài `init`, và cấp phát `mnt_alloc_id()` đơn điệu tránh đứt quãng ID trong `mountinfo`. |
| **BRENE** | [`longnguyen5512/BRENE`](https://github.com/longnguyen5512/BRENE) | `pixel5-fix` | Module ẩn môi trường/ROM + Zygisk hook trong `system_server` (`SettingsBinderProxy`, `ActivityBinderProxy`, `AccessibilityBinderProxy`) ẩn `adb_enabled`, `adb_wifi_enabled`, `development_settings_enabled` và loại bỏ `"adb"` khỏi broadcast `android.hardware.usb.action.USB_STATE` cho các app mục tiêu (hỗ trợ cả `isolated_app`). |
| **AlwaysStrong** | [`longnguyen5512/AlwaysStrong`](https://github.com/longnguyen5512/AlwaysStrong) | `main` | Fork của `Evokerr/AlwaysStrong` (TEESimulator-RS + PIF), bổ sung patch Android 17 (`vendor/teesimulator-rs-android17-keystore2.patch`) giữ nguyên chuỗi exception Rust gốc của `keystore2` (`Error::Rc(r#KEY_NOT_FOUND)`) và tối ưu `getNumberOfEntries` trong `tee_classes.dex`. |
| **SBrowserFix** | [`longnguyen5512/SBrowserFix`](https://github.com/longnguyen5512/SBrowserFix) | `main` | Module LSPosed / Vector (`com.fix.sbrowser`) ép `View.LAYER_TYPE_SOFTWARE` trên các Activity Settings của Samsung Internet để khắc phục lỗi crash `RenderThread` HWUI trên Android 17. |

---

## Tài liệu tham khảo nhanh

Xem thêm hướng dẫn cập nhật và lưu ý cấu hình từng module trên Pixel 5 tại [`PIXEL5_MODULE_UPDATE_NOTES.md`](./PIXEL5_MODULE_UPDATE_NOTES.md).

## Clone đầy đủ cả 4 repo

```bash
git clone --recursive https://github.com/longnguyen5512/pixel5-root-suite.git
```

Hoặc cập nhật submodule sau khi clone:

```bash
git submodule update --init --recursive --remote
```
