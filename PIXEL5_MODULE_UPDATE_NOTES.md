# Ghi chú sửa Module trước khi Update (Pixel 5 — KernelSU-Next + SUSFS v2.4.0)

Tài liệu này tổng hợp toàn bộ các điểm cần kiểm tra và chỉnh sửa trước khi update các module trên Pixel 5 (`redfin`, Android 17 / API 37, Kernel `4.19.325` + `KernelSU-Next v3.4.2-legacy-susfs`).

---

## 1. BRENE (`rrr333nnn333/BRENE`)

Mỗi khi tải bản BRENE mới, cần sửa 4 chỗ trong thư mục `module/` trước khi đóng gói ZIP:

1. **Tắt Hot Install trong `customize.sh`**
   - Xóa hoặc comment đoạn:
     ```bash
     if [[ "${KSU_HAS_HOT_INSTALL}" == true ]]; then
         export MODULE_HOT_INSTALL_REQUEST=true
     fi
     ```
   - **Lý do:** Hot Install chạy `post-fs-data.sh` và `boot-completed.sh` trực tiếp trên hệ thống đang chạy, dễ gây treo máy/reboot.

2. **Xóa `resetprop -c --force` trong `utils.sh` và `boot-completed.sh`**
   - Xóa dòng `resetprop -c --force` ở cuối `spoof_system_properties()` trong `utils.sh` và cuối `boot-completed.sh`.
   - **Lý do:** Ép rebuild `/dev/__properties__/*` trong lúc `init` và `system_server` đang chạy trên Android 16/17 có thể gây treo máy.

3. **Sửa cú pháp bash `< <(...)` trong `utils.sh` (`brene_clone_perm`)**
   - Thay:
     ```bash
     read -r permission owner group < <(busybox stat -c "%a %U %G" "${FROM}")
     ```
   - Thành (tương thích POSIX `/system/bin/sh`):
     ```bash
     set -- $(busybox stat -c "%a %U %G" "${FROM}")
     permission="$1"
     owner="$2"
     group="$3"
     ```

4. **Kiểm tra `config.sh` (ADB & Developer Options)**
   - Mặc định upstream đặt `config_usb_debugging=0` và `config_developer_options=0`, sẽ tự chạy `settings put global adb_enabled 0` và `development_settings_enabled 0` khi boot.
   - Đổi `config_usb_debugging=1` và `config_developer_options=1` nếu muốn giữ ADB hoạt động.

5. **Tích hợp `BRENE-Zygisk` (`zygisk/arm64-v8a.so`) ẩn `Settings.Global` trong `system_server`**
   - Source nằm tại `BRENE/zygisk-src/` (`java/com/brene/SettingsBinderProxy.java`, `BreneSettingsHook.java`, `jni/main.cpp`, `build_zygisk.sh`).
   - Chỉ hook bên trong process `system_server` (gỡ bỏ hoàn toàn khỏi app thường qua `ZYGISK_OPTION_FORCE_DENYLIST_UNMOUNT` / `dlclose`), wrap `SettingsProvider` (`IContentProvider` Binder proxy) để trả `0` cho các key `adb_enabled`, `adb_wifi_enabled`, `development_settings_enabled` khi caller là app (`uid >= 10000`) trong danh sách ẩn (`jdwp_allow_uids` / `brene_uid_list`).
   - **Lưu ý build `arm64-v8a.so`:** Không dùng `-fvisibility=hidden` và phải link `-static-libstdc++` để `zygisk_module_entry` / `zygisk_companion_entry` được export và không phụ thuộc `libc++_shared.so`.

6. **Loại trừ Framework / Overlay / Resource Cache khỏi `brene_sus_path_loop` trong `post-fs-data.sh`**
   - Trong `config_hide_custom_rom_paths` và `config_hide_custom_rom_paths_2`, chỉ gọi `brene_sus_map "${path}"` (bỏ qua `brene_sus_path_loop "${path}"`) đối với `*/framework/*`, `*/overlay/*`, `*-res.apk`, `*platform-res*`, `/data/resource-cache/*`, `*overlay*`, `*idmap*`.
   - **Lý do:** Chặn `open()` lên `/system/framework/org.lineageos.platform-res.apk`, `/system_ext/overlay/LineageFrameworksOverlay.apk` hoặc `.idmap` sẽ làm `app_zygote` (như `detector_zygote`) crash `JNI FatalError` khi preload tài nguyên, gây cảnh báo `SELinux` (`Enforcing with app_zygote carrier startup failure`).

7. **Sync `adb_tls_port` & `jdwp_allow_uids` (`utils.sh` / `boot-completed.sh`)**
   - `apply_usb_debug_allow_apps` đồng bộ `jdwp_allow_uids` và cổng Wireless Debugging (`adb_tls_port`) xuống kernel (`/sys/module/unix/parameters/adb_tls_port`) để chặn `connect()` và ẩn socket trong `/proc/net/tcp*`.

8. **WebUI React + Vite + Tailwind + M3 (`src/` → `module/webroot/`)**
   - Đã migrate WebUI sang React + Vite + Tailwind CSS + Material 3 (`npm run build` xuất ra `module/webroot/index.html` và `module/webroot/assets/`).
   - Khi giải nén hoặc cập nhật thủ công vào `/data/adb/modules/brene/`, nhớ `chmod 755 /data/adb/modules/brene/tools/* /data/adb/modules/brene/*.sh` để giữ quyền thực thi cho `tools/susfs`.

9. **Giữ `ro.build.fingerprint` khớp với `Build.FINGERPRINT` trong `boot-completed.sh`**
   - Không ghi đè `ro.build.fingerprint` / `ro.*.build.fingerprint` / `ro.build.description` theo `ro.build.version.release` + `ro.build.id` (`17/CP2A...`), vì `/system/build.prop` và hằng số `Build.FINGERPRINT` trong framework dùng fingerprint stock Android 14 (`google/redfin/redfin:14/UP1A.231105.001.B2/11260668:user/release-keys`). Ghi đè sẽ gây cảnh báo `ro.build.fingerprint <> Build.FINGERPRINT: Drift` trên Duck Detector.

> **Lưu ý:** Nếu cài BRENE qua `ksud` CLI bị timeout cảnh báo bảo mật (chờ bấm Volume Down 5s), tạm tắt rồi bật lại:
> `ksud module risk disable && ksud module install BRENE.zip && ksud module risk enable`

---

## 2. NeoZygisk (`JingMatrix/NeoZygisk`)

1. **Kiểm tra `MAX_KSU_VERSION` trong `customize.sh`**
   - Dùng bản build từ nhánh `master` mới (như `v2.4-301-cc0bbba` trở lên) đã gỡ `MAX_KSU_VERSION`.
   - Nếu gặp bản cũ có `MAX_KSU_VERSION=30000` (gây cảnh báo `KernelSU version too large!` với KSU-Next `33314+`), nâng lên `50000` (hoặc gỡ check) **và nhớ cập nhật lại `customize.sh.sha256`** (đúng 64 ký tự hex, không xuống dòng) để `verify.sh` không fail.

2. **KHÔNG xóa `allow system_server system_server process execmem` trong `sepolicy.rule`**
   - **Lý do:** HMA-OSS cần `mprotect(..., PROT_EXEC)` trong `system_server`; thiếu rule này sẽ crash `EACCES (Permission denied)`.
   - Việc ẩn rule này trước Duck Detector (`DirtySepolicy`) đã được xử lý ở tầng kernel (`drivers/kernelsu/feature/selinux_hide.c`, commit `3bfc33850868`).

---

## 3. HMA-OSS (`frknkrc44/HMA-OSS`)

1. **Thứ tự cài:** Luôn cài **sau** `NeoZygisk` để `customize.d/22-check-zygisk.sh` phát hiện được `zygiskd`.
2. **Checksum khi tối ưu ABI:** Nếu xóa `zygisk/x86.so`, `x86_64.so`, `armeabi-v7a.so` khỏi ZIP, phải xóa các dòng `do_verify` tương ứng trong `customize.d/00-verify-resources.sh`.
3. **Cấu hình sau cài:** Mở app HMA-OSS, kiểm tra `Work mode` hoạt động bình thường (không có lỗi `mprotect failed: EACCES` trong log), tạo template ẩn các app root/manager cho các app cần che (như `com.eltavine.duckdetector`).

---

## 4. AlwaysStrong [Inject] (`Evokerr/AlwaysStrong`)

1. **Fetch Keybox & PIF sau mỗi lần update/reboot:**
   - Bấm **[Action]** trong KernelSU-Next Manager (hoặc chạy `ksud module action tricky_store`) để cập nhật `keybox.xml` mới thay cho keybox AOSP mặc định (bị `CertificateExpiredException`).
2. **Kiểm tra `/data/adb/tricky_store/boot_key.bin`:**
   - Nếu `od -tx1 /data/adb/tricky_store/boot_key.bin` toàn `00` (gây lỗi `Attested verifiedBootKey was all zeros`), ghi lại 32 byte ngẫu nhiên khác `0`:
     ```bash
     dd if=/dev/urandom of=/data/adb/tricky_store/boot_key.bin bs=32 count=1
     chmod 644 /data/adb/tricky_store/boot_key.bin
     ```
3. **Kiểm tra `/data/adb/tricky_store/target.txt`:** Đảm bảo có tên package các app cần spoof attestation (ví dụ `com.eltavine.duckdetector`).

---

## 5. Vector / LSPosed (`JingMatrix/Vector`)

- Dùng **JingMatrix/Vector** (`zygisk_vector`, ví dụ `Vector-v2.2-3080-Release.zip`) thay cho các bản LSPosed fork cũ (`LSPosed-v2.2.1.zip` bị crash `SIGSEGV pc=0x314` trên Android 17 + NeoZygisk).
- Đảm bảo `config_hide_vector_injection=1` được bật trong BRENE để SUSFS ẩn `zygisk_vector` map.

---

## 6. Kernel (`306kernels-redbull`, branch `3.4.2-dev`)

Khi rebase/update kernel mới, đảm bảo giữ lại các bản vá sau:
1. `fs/susfs.c`: Kiểm tra `if (old_bitmap)` trước cả 4 lệnh `kfree_rcu(old_bitmap, rcu);` (tránh kernel panic NULL pointer dereference trên Linux `4.19` khi chạy `set_priority_uid_filter`).
2. `drivers/kernelsu/feature/selinux_hide.c`, `selinux_hide.h`, `security/selinux/hooks.c`, `security/selinux/selinuxfs.c`:
   - Che bit `process:execmem` (`1U << 25`) của `u:r:system_server:s0` (class `2`) đối với app `UID >= 10000` khi query `/sys/fs/selinux/access`.
   - Ẩn các type/rule nhạy cảm (`xposed_file`, `adbroot`, `kernel -> adb_data_file`, `dex2oat -> dex2oat_exec`), chặn probe `/sys/fs/selinux/access` (`-EINVAL`), và cân bằng timing `sel_write_context` / `selinux_setprocattr` (lookup `u:r:app_zygote:s0`).
3. `fs/proc/task_mmu.c`: Làm sạch `mss` (`Shared_Dirty`, `Private_Dirty`, `Anonymous`) của `webview.apk` mapping `r-xp` đối với `uid >= 10000`.
4. `net/unix/af_unix.c`, `net/ipv4/af_inet.c`, `net/ipv4/tcp_ipv4.c`, `net/ipv6/tcp_ipv6.c`: Thêm `/sys/module/unix/parameters/adb_tls_port` + `ksu_is_adb_hidden_port(u16 port)` dùng `jdwp_allow_uids` để chặn `connect()` và ẩn port Wireless/USB ADB (`adb_tls_port`, `5555`) khỏi `/proc/net/tcp*`. Đổi quyền `module_param` `jdwp_allow_uids` và `adb_tls_port` từ `0644` sang `0600`.
5. `drivers/kernelsu/runtime/boot_event.c`, `ksud_integration.c`: Gọi `stop_init_rc_hook()` và tắt `ksu_vfs_read_hook` tại `on_post_fs_data()`.
6. `fs/proc/version.c`: Spoof `/proc/version` theo `susfs` cho `uid >= 10000`.
7. `fs/hybridmount/`, `fs/Kconfig`, `fs/Makefile`, `arch/arm64/configs/redbull_defconfig`, `arch/arm64/configs/redbull-gki_defconfig`: Tích hợp Hybrid Mount VFS built-in (`CONFIG_HYBRIDMOUNT=y`, giao thức `hm1`) từ `Hybrid-Mount/meta-hybrid_mount` (`module/vfs/setup.sh`) để `hybrid_mount` sử dụng VFS kernel trực tiếp (`/sys/module/hybridmount`).

---

## 7. RomFeatureHide (`longnguyen5512/romfeature-hide`)

1. **Scope:** Bật trong Vector / LSPosed Manager với scope **System Framework** (`android`), sau đó reboot.
2. **An toàn `getPackagesForUid`:** Hook `IPackageManagerBase.getPackagesForUid` phải bỏ qua khi `uid == Binder.getCallingUid()` và trả `null` (tuyệt đối không trả `new String[0]`) khi danh sách sau lọc rỗng — nếu không, các app hệ thống (`org.lineageos.glimpse`, `org.lineageos.audiofx`, `org.lineageos.recorder`, `org.omnirom.logcat`) sẽ crash `ArrayIndexOutOfBoundsException` trong `Settings.getPackageNameForUid()`.

---

## 8. SBrowserFix (`longnguyen5512/SBrowserFix`)

- Cài `SBrowserFix.apk`, bật trong Vector / LSPosed Manager với scope **Samsung Internet** (`com.sec.android.app.sbrowser`) để ép `View.LAYER_TYPE_SOFTWARE` trên các Activity Settings, khắc phục crash `RenderThread` HWUI trên Android 17.

---

## 9. Pixelify-Next (`longnguyen5512/Pixelify-Next`, nhánh `DEV`)

1. **Loại trừ GMS / Play Store / DuckDetector trong `beta/zygisk/src/main/cpp/pixelify.cpp`:**
   - Thêm `com.google.android.gms`, `com.android.vending`, `com.eltavine.duckdetector` vào `isExcluded()` để không spoof thành Pixel 2/6 Pro/8 Pro và không xung đột với `AlwaysStrong [Inject]`.
2. **Cấu hình `beta/module/config.prop`:**
   - Đặt `GLOBAL_SPOOFING=0` (không ghi đè `ro.build.fingerprint` / `ro.product.model` toàn cục), `ENABLE_DPS=0` (Pixel 5 đã có `com.google.android.as` từ NikGapps), `QUICK_SHARE=1` (tích hợp `mosey-p8a`).
3. **Build cho Pixel 5 (`arm64-v8a`):**
   - Dùng bản `PixelifyNext-BETA-v7.1-no-VK.zip` khi cài qua `ksud` (đã tối ưu `arm64-v8a` only trong `beta/build.gradle` và sửa `sed -i` tương thích macOS trong `scripts/build_mosey.sh`).

---

## 10. mosey-p8a (`longnguyen5512/mosey-p8a`, nhánh `main`) — Quick Share AirDrop

1. **`wonder_mosey_wild.ko` (`cfg80211`, kernel `4.19.325`):**
   - Kernel `redbull` bật `CONFIG_CFG80211=y` và tắt `CONFIG_MAC80211`, nên `wonder/wonder_mosey_wild.c` dùng trực tiếp `wiphy_new_nm(..., "wonder")` + `wiphy_register()` và biên dịch theo `306kernels-redbull`. Cập nhật `customize.sh` cho phép `4.19.*`.
2. **Patch `mosey_server` cho ARMv8.2-A:**
   - Snapdragon 765G không có ARMv8.3 PAC; phải patch 1619 lệnh `retaa` (`0xd65f0bff`) → `ret` (`0xd65f03c0`) trong `system/vendor/bin/mosey_server` để tránh `SIGILL`.
3. **`sepolicy.rule` chuẩn KernelSU (`ksud sepolicy`):**
   - Không dùng `;`, không dùng `:` giữa target/class, không dùng `self` (thay bằng `mosey_server`), không để khoảng trắng đầu/cuối trong `{...}`, và thêm `allow ksu mosey_service service_manager {add find}`.
4. **`mosey_vintf_patch` (lọc riêng `com.google.pixel.moseyservice`):**
   - `servicemanager` cache Device HAL Manifest trước `post-fs-data` và `/vendor` không còn trống để chép `manifest_mosey.xml`. `wonder/mosey_vintf_patch.c` patch in-memory check `hasAidlInstance` trong `servicemanager` **chỉ** cho `com.google.pixel.moseyservice` (29 ký tự, prefix `com.goog`).
   - **Cảnh báo:** Tuyệt đối không trả `true` cho mọi HAL vì sẽ làm `libgui` chờ `android.hardware.graphics.allocator.IAllocator/default` (AIDL) vô hạn thay vì fallback HIDL `@4.0`, gây ANR Google Play Services.
5. **`service.sh` & `customize.sh`:**
   - Cấp quyền `0755` cho `sqlite3_mosey`, `rename_phy`, `mosey_vintf_patch` trong `customize.sh`; kiểm tra `/proc/$oldpid/cmdline` chứa `mosey_server` trước khi skip theo `mosey_server.pid` và chạy `mosey_vintf_patch` trước khi start `mosey_server`.

---

## 11. vneid_persist (`longnguyen5512/vneid_persist`, nhánh `main`)

1. **Cơ chế hoạt động (`v2.6`):**
   - `prop_watch.sh` dùng `bin/resetprop-rs --wait` (ngủ trên futex, 0% CPU) theo dõi `sys.lineage_settings_{system,secure,global}_version` và `--nuke` ngay khi prop tái sinh; đồng thời giữ FD mở (`busybox tail -n 0 -f`) để ghim inode `open_redirect` cho `*sepolicy.cil`, `*file_contexts`, `/system_ext/etc/permissions/Updater.xml`.
   - `action.sh` hỗ trợ xoá cache verdict cũ của VNeID (`cmwoawp.ogg`, `xwoccmwldwasxm.dat`, `cache/*`, `app_webview/*`) mà vẫn giữ nguyên `SKS_*` (đăng nhập/sinh trắc).
2. **Đã tối ưu cho Pixel 5 (`redfin`):**
   - Đã gỡ bỏ đoạn ghi đè `ro.build.flavor=unicorn-user` và `ro.build.fingerprint` (Xiaomi `unicorn`) trong `post-fs-data.sh` / `service.sh` để giữ `ro.build.fingerprint` khớp với `Build.FINGERPRINT` khi chạy cùng **BRENE**.
   - Tự động nhận diện `/data/adb/ksu/bin/ksu_susfs` hoặc `/data/adb/ksu/bin/susfs` trong `service.sh` và `prop_watch.sh`.
   - Loại trừ `*/framework/*`, `*/overlay/*`, `*-res.apk`, `*idmap*`, `/data/resource-cache/*` khỏi `sus_path` / `sus_path_loop` (chỉ dùng `sus_map`) để `AppZygote` không crash.
