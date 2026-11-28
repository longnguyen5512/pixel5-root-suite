# Pixel 5 (`redfin`) Android 17 Root, Hardening & Pixel Features Suite

Bộ tổng hợp repository, bản vá kernel, module Zygisk/LSPosed và hướng dẫn cấu hình hoàn chỉnh để ẩn root/custom ROM ổn định qua reboot, giữ Play Integrity / Duck Detector sạch và bật tính năng độc quyền (bao gồm Quick Share AirDrop) trên **Google Pixel 5 (`redfin`)**.

- **Thiết bị:** Google Pixel 5 (`redfin`), Snapdragon 765G (`ARMv8.2-A`, không có ARMv8.3 PAC)
- **ROM:** crDroid 13.1 / LineageOS 24 base (Android 17, API 37), `arm64-v8a`
- **Root:** KernelSU-Next v3.4.2-legacy-susfs + SUSFS + NeoZygisk + Hybrid Mount
- **Kernel tuỳ biến:** [`306kernels-redbull`](https://github.com/longnguyen5512/306kernels-redbull) (`4.19.325-cip136-st20`, nhánh `3.4.2-dev`)

---

## 0. Cấu trúc Suite & Repository thành phần

```
pixel5-root-suite/
├── README.md                        ← tài liệu tổng hợp & hướng dẫn cài đặt
├── PIXEL5_MODULE_UPDATE_NOTES.md    ← checklist chi tiết trước khi update từng module
├── 306kernels-redbull/              ← Kernel 4.19.325 + KSU-Next + SUSFS + Hybrid Mount VFS + SELinux/proc/af_unix patches
├── BRENE/                           ← Module ẩn ROM/root + Zygisk hook trong system_server
├── AlwaysStrong/                    ← Fork TEESimulator-RS + PIF (patch Android 17 keystore2)
├── SBrowserFix/                     ← Module LSPosed/Vector fix crash RenderThread Samsung Internet
├── RomFeatureHide/                  ← Module LSPosed/Vector ẩn feature/permission/package custom ROM
├── Pixelify-Next/                   ← Module bật tính năng Pixel độc quyền (nhánh DEV, tối ưu arm64-v8a)
├── mosey-p8a/                       ← Module Quick Share AirDrop (wonder phy cfg80211 + mosey_vintf_patch)
└── vneid_persist/                   ← Module dọn prop lineage/adb_root (resetprop-rs) + VFS-pinned open_redirect
```

| Thành phần | Repository | Nhánh chính | Vai trò |
|---|---|---|---|
| **Kernel** | [`longnguyen5512/306kernels-redbull`](https://github.com/longnguyen5512/306kernels-redbull) | `3.4.2-dev` | Kernel `4.19.325` + KernelSU-Next + SUSFS + Hybrid Mount VFS (`CONFIG_HYBRIDMOUNT=y`); vá timing `selinux_hide` (2-slot double-hash không nhánh), tách `get_anon_bdev()`, cấp phát `mnt_alloc_id()` đơn điệu, dừng `init_rc_hook`/`vfs_read_hook` sau `post-fs-data`, spoof `/proc/version`, và đặt quyền `0600` cho `jdwp_allow_uids`/`adb_tls_port`. |
| **BRENE** | [`longnguyen5512/BRENE`](https://github.com/longnguyen5512/BRENE) | `pixel5-fix` | Ẩn đường dẫn/prop custom ROM + Zygisk hook trong `system_server` (`SettingsBinderProxy`, `ActivityBinderProxy`, `AccessibilityBinderProxy`) ẩn `adb_enabled`, `adb_wifi_enabled`, `development_settings_enabled`, lọc `"adb"` khỏi broadcast `USB_STATE` và giữ `ro.build.fingerprint` khớp với `Build.FINGERPRINT`. |
| **AlwaysStrong** | [`longnguyen5512/AlwaysStrong`](https://github.com/longnguyen5512/AlwaysStrong) | `main` | Fork của `Evokerr/AlwaysStrong` (TEESimulator-RS + PIF), bổ sung patch Android 17 (`vendor/teesimulator-rs-android17-keystore2.patch`) giữ nguyên chuỗi exception Rust gốc của `keystore2` (`Error::Rc(r#KEY_NOT_FOUND)`). |
| **SBrowserFix** | [`longnguyen5512/SBrowserFix`](https://github.com/longnguyen5512/SBrowserFix) | `main` | Module LSPosed / Vector (`com.fix.sbrowser`) ép `View.LAYER_TYPE_SOFTWARE` trên các Activity Settings của Samsung Internet để khắc phục lỗi crash `RenderThread` HWUI trên Android 17. |
| **RomFeatureHide** | [`longnguyen5512/romfeature-hide`](https://github.com/longnguyen5512/romfeature-hide) | `main` | Module LSPosed / Vector (`com.fix.romfeatures`) ẩn feature, permission và package custom ROM (`org.lineageos.*`, `org.omnirom.*`, ...) khỏi app bên thứ ba trong `system_server`, đồng thời giữ nguyên lookup cho chính UID của app đang khởi động. |
| **Pixelify-Next** | [`longnguyen5512/Pixelify-Next`](https://github.com/longnguyen5512/Pixelify-Next) | `DEV` | Fork của `BasGame1/Pixelify-Next`; loại trừ `gms`/`vending`/`duckdetector` khỏi Zygisk spoof để không xung đột `AlwaysStrong`, tắt `GLOBAL_SPOOFING` và `ENABLE_DPS`, build `arm64-v8a` only kèm `mosey-p8a`. |
| **mosey-p8a** | [`longnguyen5512/mosey-p8a`](https://github.com/longnguyen5512/mosey-p8a) | `main` | Fork của `BasGame1/mosey-p8a`; chuyển `wonder_mosey_wild.c` sang `cfg80211` cho kernel `4.19.325`, patch 1619 lệnh `retaa` → `ret` trong `mosey_server`, chuẩn hoá `sepolicy.rule` cho `ksud`, và thêm `mosey_vintf_patch` lọc riêng `com.google.pixel.moseyservice`. |
| **vneid_persist** | [`longnguyen5512/vneid_persist`](https://github.com/longnguyen5512/vneid_persist) | `main` | Module KernelSU-Next + SuSFS (`v2.6`) dùng `resetprop-rs --wait` (0% CPU futex) tự `--nuke` các prop `sys.lineage_settings_*_version` tái sinh, giữ inode `open_redirect` cho `sepolicy.cil`/`file_contexts`/`Updater.xml` và kèm Action xoá cache verdict của VNeID. |

### Tự build lại Kernel (`306kernels-redbull`)

Dùng container build trong `306kernels-redbull-docker` (trên macOS Apple Silicon / Linux):

```bash
./306kernels-redbull-docker/build.sh
```

File ZIP AnyKernel3 xuất ra tại `306kernels-redbull-docker/out/306kernels-redbull-*.zip`, flash qua Recovery:

```bash
adb reboot sideload
adb wait-for-sideload
adb sideload 306kernels-redbull-docker/out/306kernels-redbull-*.zip
```

### Tự build module LSPosed / Vector (`SBrowserFix`, `RomFeatureHide`)

Yêu cầu Android SDK Build-Tools `37.0.0` và platform `android-37.2`:

```bash
cd RomFeatureHide && ./build.sh   # xuất RomFeatureHide.apk
cd SBrowserFix && ./build.sh      # xuất SBrowserFix.apk
```

---

## 1. Điểm vá & tối ưu cốt lõi

1. **Kernel (`306kernels-redbull`, `3.4.2-dev`)**:
   - `fs/susfs.c`: kiểm tra `if (old_bitmap)` trước 4 lệnh `kfree_rcu(old_bitmap, rcu)` để tránh kernel panic `NULL` dereference trên Linux `4.19`.
   - `drivers/kernelsu/feature/selinux_hide.c` + `security/selinux/*`: che bit `process:execmem` của `u:r:system_server:s0` cho `uid >= 10000` (giữ HMA-OSS chạy ổn định mà không lộ `DirtySepolicy`), ẩn `xposed_file`/`adbroot`, cân bằng timing `selinux_setprocattr`.
   - `drivers/kernelsu/runtime/boot_event.c` + `ksud_integration.c`: tắt `ksu_init_rc_hook` và `ksu_vfs_read_hook` ngay tại `on_post_fs_data()`.
   - `fs/proc/version.c` & `net/unix/af_unix.c`: spoof `/proc/version` theo `susfs` cho `uid >= 10000`, đổi quyền `jdwp_allow_uids`/`adb_tls_port` sang `0600` và ẩn port ADB trong `/proc/net/tcp*`.
   - `fs/hybridmount/` + `redbull*_defconfig`: tích hợp Hybrid Mount VFS built-in (`CONFIG_HYBRIDMOUNT=y`, `hm1`) giúp `hybrid_mount` hoạt động trực tiếp ở tầng VFS trong kernel (`/sys/module/hybridmount`).
2. **BRENE (`pixel5-fix`)**:
   - Không dùng `Hot Install` và `resetprop -c --force` (tránh treo máy trên Android 17).
   - Loại trừ `*/framework/*`, `*/overlay/*`, `*-res.apk`, `*idmap*`, `/data/resource-cache/*` khỏi `brene_sus_path_loop` (chỉ dùng `brene_sus_map`) để `app_zygote` không crash `JNI FatalError`.
   - Giữ nguyên `ro.build.fingerprint` gốc khớp với `Build.FINGERPRINT` (tránh cảnh báo `ro.build.fingerprint <> Build.FINGERPRINT: Drift` trên Duck Detector).
3. **RomFeatureHide (`com.fix.romfeatures`)**:
   - Hook `getPackagesForUid` bỏ qua lọc khi `uid == Binder.getCallingUid()` và trả `null` (thay vì `String[0]`) khi rỗng, giúp các app hệ thống (`org.lineageos.glimpse`, `org.lineageos.audiofx`, `org.lineageos.recorder`, `org.omnirom.logcat`) không bị `ArrayIndexOutOfBoundsException` trong `Settings.getPackageNameForUid()`.
4. **Quick Share AirDrop (`Pixelify-Next` / `mosey-p8a` cho Pixel 5)**:
   - `wonder_mosey_wild.c`: chuyển từ `mac80211` (không bật trong kernel `redbull`) sang `cfg80211` (`wiphy_new_nm(..., "wonder")` + `wiphy_register()`), biên dịch khớp kernel `4.19.325`.
   - `mosey_server`: patch 1619 lệnh ARMv8.3 PAC `retaa` → `ret` để chạy được trên Snapdragon 765G (`ARMv8.2-A`) không bị `SIGILL`.
   - `mosey_vintf_patch`: patch in-memory check `hasAidlInstance` trong `/system/bin/servicemanager` chỉ riêng cho `com.google.pixel.moseyservice` (giữ nguyên các HAL khác như `IAllocator@4.0`, tránh ANR Google Play Services).

---

## 2. Thứ tự cài đặt & Cấu hình chuẩn

1. **Flash Kernel** (`306kernels-redbull-*.zip`) qua `adb sideload`.
2. **Metamodule & Zygisk**:
   - Cài `Hybrid Mount` (hoặc metamodule mount tương đương).
   - Cài **NeoZygisk** (`zygisksu`) trước các module Zygisk khác.
3. **LSPosed / Vector & Ẩn App**:
   - Cài **Vector** (`zygisk_vector`, ví dụ `v2.2-3080`) — bật `config_hide_vector_injection=1` trong BRENE.
   - Cài **HMA-OSS** (sau NeoZygisk), tạo template ẩn root/Xposed cho app cần che (như `com.eltavine.duckdetector`).
   - Cài **RomFeatureHide.apk** (bật scope **System Framework** `android`) và **SBrowserFix.apk** (bật scope **Samsung Internet**), sau đó reboot.
4. **BRENE & AlwaysStrong**:
   - Cài **BRENE** (`ksud module risk disable && ksud module install BRENE.zip && ksud module risk enable`).
   - Cài **AlwaysStrong [Inject]** (`tricky_store`), chạy Action để fetch `keybox.xml` + `pif.prop`, kiểm tra `/data/adb/tricky_store/boot_key.bin` không toàn `00` và thêm package cần attest vào `target.txt`.

> Xem thêm lưu ý chi tiết cho từng module tại [`PIXEL5_MODULE_UPDATE_NOTES.md`](./PIXEL5_MODULE_UPDATE_NOTES.md).

---

## 3. Lưu ý & Xử lý sự cố thường gặp

- **Cảnh báo `RKP map[3]=nubia` trên Duck Detector (`[✖ DANGER] TEE`)**: do `keybox.xml` công khai từ AlwaysStrong hiện là RKP cert của máy Nubia Tablet ký bởi `Droid CA3` (không thể sửa nội dung cert mà không hỏng chữ ký). Cần thay bằng keybox Google/Pixel riêng (bật `custom_keybox=1`) nếu muốn xử lý triệt để.
- **Lệch `ro.build.fingerprint <> Build.FINGERPRINT`**: đảm bảo không bật global spoofing của các module khác ghi đè `ro.build.fingerprint` khác với `Build.FINGERPRINT` trong framework.
- **Google Play Services treo khi dùng `mosey_vintf_patch`**: tuyệt đối không bypass toàn bộ `hasAidlInstance` trong `servicemanager` (sẽ làm `libgui` chờ `IAllocator` AIDL vô hạn); luôn dùng bản `mosey_vintf_patch` chỉ lọc `com.google.pixel.moseyservice`.

---

## 4. Clone & Cập nhật

Clone đầy đủ cả 8 repository thành phần:

```bash
git clone --recursive https://github.com/longnguyen5512/pixel5-root-suite.git
```

Hoặc cập nhật submodule sau khi clone:

```bash
git submodule update --init --recursive --remote
```
