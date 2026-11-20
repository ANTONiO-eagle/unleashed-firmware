# Unleashed F7: локальная проверка LTO · 20 ноября 2026

**Статус: неофициальный эксперимент ANTONiO-eagle, не релиз Unleashed и не рекомендация к установке.** Это материалы из личного форка, не одобренные DarkFlippers или Flipper Devices. Прошиваемых файлов здесь нет; изменения не внесены в upstream и не применены к этой ветке. Правки ниже — текст для воспроизведения замера.

## База и разделение экспериментов

- [Unleashed `dev` / `7f31d0cca51302fd103232a7e09705a0a5840429`](https://github.com/DarkFlippers/unleashed-firmware/commit/7f31d0cca51302fd103232a7e09705a0a5840429), F7, API 88.27. Коммиты 16 подмодулей в `locked-submodules.txt`. Штатный Linux x86_64 FBT toolchain 39, Arm GNU GCC 12.3.1 20230626, Binutils 2.40.0.20230627.
- В базовом FBT release-пути `-flto` не было. Опытный `FW_LTO=1` по умолчанию выключен и добавляет только `-flto=1` к компиляции встроенного C/C++ и финальной линковке основной прошивки при `COMPACT=1 DEBUG=0`. Статические библиотеки, флаги внешних FAP и отдельный RAM updater не меняются. Базовые `-Os`, `--gc-sections`, `nano.specs`, набор приложений и API остаются прежними.
- Первый проход на штатных исходниках: **875 600 → 868 676 Б (−6 924 Б; −0,7908%)**. Он подтвердил экономию, но также обнаружил предупреждения. Его нельзя использовать как рекомендацию установить сборку или включить LTO по умолчанию. Ранее измеренные −7 356 Б — результат отдельной сборки Hybridus; не относится к этой базе.

## Диагностика и локальные исправления

Сгенерированный FBT `applications.c` (генератор `scripts/fbt_tools/fbt_apps.py:51–54`) объявляет startup hooks как `void(void)`, что соответствует `FlipperInternalOnStartHook` и вызову без аргументов из `applications/services/loader/loader.c:1076–1078`. Шесть определений изначально не соответствовали контракту:

| Исходник (строка на зафиксированной базе) | Символ | Было |
|---|---|---|
| `applications/services/rpc/rpc.c:441` | `rpc_on_system_start` | `void(void*)` |
| `applications/services/gps/gps.c:86` | `gps_on_system_start` | `void(void*)` |
| `applications/services/network/network.c:162` | `network_on_system_start` | `void(void*)` |
| `applications/services/expansion/expansion.c:292` | `expansion_on_system_start` | `void(void*)` |
| `applications/services/locale/locale.c:109` | `locale_on_system_start` | `int32_t(void*)` |
| `applications/services/namechanger/namechanger.c:73` | `namechanger_on_system_start` | `int32_t(void*)` |

Поиск ссылок показал, что параметры не читались, результаты двух функций не использовались, а символы не отмечены в SDK API. `startup-contract.patch` приводит определения к контракту, не меняя последовательность вызовов или условия выхода. `check_hooks_source.py` проверяет это по исходникам. Межмодульная диагностика LTO **обнаружила**, а не создала несовместимость типов; в C вызов через несовместимое объявление имеет неопределённое поведение.

В `applications/settings/storage_settings/storage_settings.c:3,57–61` `memcpy` запрашивал 56 Б от `descriptor_template`, у типа которого размер заголовка 8 Б. Оба исходных ELF действительно содержат 56-байтный символ (§ [расширение GNU C для статической инициализации гибкого массива](https://gcc.gnu.org/onlinedocs/gcc-12.2.0/gcc/Zero-Length.html)); поэтому `-Wstringop-overread` **не доказывает фактическое чтение за выделенным объектом** в этих образах. Исходный способ копирования зависел от расширения и вызвал диагностику при межмодульной оптимизации. `storage-descriptor.patch` переносит неизменные шесть записей в массив явного размера, инициализирует заголовок по полям и копирует только массив в выделенную изменяемую часть. API и helper не затронуты. Изолированный Cortex-M Thumb-тест на Unicorn 2.1.4 проверил 56-байтное выделение, шесть строк и ID сцен, оба состояния Mount/Unmount с LTO и без. Это не эмуляция всего устройства.

## Чистый A/B после исправлений

Обе сборки запущены с нуля (без `build`, базы SCons и кэша компилятора) с одинаковыми правками. Оба прохода завершились с кодом 0. Встроенный C/C++ и финальный линковщик отличаются только `-flto=1`; сверены 830 команд компиляции, LTO добавлен в 227 из них. Команды 19 объектов Desktop Settings FAP не изменились. Во всех 32 статических архивах совпали порядок, содержимое членов и индекс; в сырых архивах отличаются только временные метки заголовков `ar`. Их сверка не меняет собранные образы.

| Артефакт | Без LTO | С LTO | Дельта |
|---|---:|---:|---:|
| BIN / прошиваемые данные DFU | 875 616 Б | 868 692 Б | **−6 924 Б** |
| DFU целиком | 875 925 Б | 869 001 Б | **−6 924 Б** |
| Страницы по 4 КиБ, затронутые диапазоном BIN | 214 | 213 | −1 |

Экономия **0,7908%** от BIN. В каждом DFU один элемент по адресу `0x08000000`: его данные побайтно совпадают с BIN, CRC корректна; контейнер добавляет 309 Б. Без LTO исправления сами по себе добавляют 16 Б к штатному BIN (875 600 → 875 616 Б). Метаданные прошивок совпали. Проверены все **3 046 ключей API 88.27** и допустимые диапазоны их адресов, все **14 startup hooks** и их порядок, одинаковый размер `.bss` (7 544 Б), совпадение Desktop Settings FAP побайтно (40 152 Б, SHA-256 `c88ecb099357b1af36fa52917dfaecc86a450fb970044d33f191a60e96e5908f`).

Шесть `-Wlto-type-mismatch` и `-Wstringop-overread` исчезли без отключения диагностик. Единственное оставшееся предупреждение LTO-прохода — о последовательном выполнении 7 этапов LTRANS (`lto-wrapper: warning: using serial compilation of 7 LTRANS jobs`); оно относится к режиму работы компилятора, не к корректности кода.

## Штатный профиль `unit_tests` и границы проверки

| Проверка | Без LTO | С LTO |
|---|---|---|
| Штатный `updater_package` с `FIRMWARE_APP_SET=unit_tests` | Код выхода 0; 30 тестовых плагинов, 131 файл тестовых данных побайтно совпал с исходниками; 627 уникальных импортов сверены с API | Остановлен вручную на линковке прошивки и сборке RAM-обновлятора; пакет не получен |
| `varint` на ARM-эмуляторе | 4 случая, 800 028 утверждений, 0 сбоев | 4 случая, 800 028 утверждений, 0 сбоев |
| `datetime` на ARM-эмуляторе | 19 случаев, 25 утверждений, 0 сбоев | 19 случаев, 25 утверждений, 0 сбоев |
| Полный `unit_tests` на реальном F7 | Не запускался | Не запускался |

Штатная цель: `./fbt -j 2 COMPACT=1 DEBUG=0 LIB_DEBUG=0 TARGET_HW=7 FIRMWARE_APP_SET=unit_tests FW_LTO=0 updater_package`; для второго плеча — `FW_LTO=1`. Это другой профиль, не образы замера из таблицы выше. В эмуляции использованы проверки из репозитория и объекты `varint.o` и `datetime.o` двух сборок; вспомогательная обвязка подменила вывод и генератор случайных чисел. Проверка импортов статическая и не подтверждает работу загрузчика или плагинов.

**Что осталось:** завершить штатный пакет с LTO, выполнить [полный `unit_tests`](https://github.com/DarkFlippers/unleashed-firmware/blob/7f31d0cca51302fd103232a7e09705a0a5840429/documentation/UnitTests.md) на тестовом F7 и проверить запуск и работу обоих образов A/B на устройстве. Замер размера завершён; готовность прошивки не подтверждена. Независимого повтора сборки на другом хосте не было.

## Материалы для воспроизведения

На Linux x86-64 сохраните десять файлов из блоков ниже в один каталог. Для чистого замера запустите `bash repro-fixed.sh`. Скрипт получает закреплённый коммит и подмодули, применяет общие правки и собирает обе конфигурации `firmware_dfu` и `fap_desktop_settings`; ничего не прошивает и не отправляет. Нужны Git, Python 3, curl, tar и несколько ГБ места. Патчи содержат производные фрагменты исходников Unleashed; см. [лицензию исходной базы](https://github.com/DarkFlippers/unleashed-firmware/blob/7f31d0cca51302fd103232a7e09705a0a5840429/LICENSE).

<details>
<summary><code>repro-fixed.sh</code></summary>

```bash
#!/usr/bin/env bash
# Run from an unpacked evidence package. Does not flash or publish anything.
set -euo pipefail
here=$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd -P)
repo="$here/reproduction-src"
commit=7f31d0cca51302fd103232a7e09705a0a5840429
[[ ! -e "$repo" ]] || { echo "Refusing to overwrite $repo" >&2; exit 1; }
git init -q "$repo"
git -C "$repo" remote add origin https://github.com/DarkFlippers/unleashed-firmware.git
git -C "$repo" fetch --depth 1 origin "$commit"
git -C "$repo" checkout -q -b dev "$commit"
git -C "$repo" submodule update --init --recursive --depth 1 --jobs 4
cd -- "$repo"
git submodule status --recursive | awk '{print $1, $2}' > "$here/reproduction-submodules.txt"
diff -u <(awk '{print $1, $2}' "$here/locked-submodules.txt") "$here/reproduction-submodules.txt"
git apply "$here/prerequisites.patch" "$here/experimental-fw-lto.patch"
git diff --check
export TZ=UTC FBT_NO_SYNC=1 FBT_VERBOSE=1 FORCE_NO_DIRTY=1 DIST_SUFFIX=lto-ab-fixed CCACHE_DISABLE=1
export SOURCE_DATE_EPOCH=$(git log -1 --format=%ct)
export FBT_TOOLCHAIN_VERSION=39
unset FBT_NOENV FBT_TOOLCHAIN_PATH FBT_TOOLS_CUSTOM_LINK WORKFLOW_BRANCH_OR_TAG
for mode in baseline candidate; do
  opt=0
  [[ $mode != candidate ]] || opt=1
  [[ ! -e build && ! -e .sconsign.dblite ]] || exit 3
  ./fbt -j "${JOBS:-2}" COMPACT=1 DEBUG=0 LIB_DEBUG=0 TARGET_HW=7 FIRMWARE_APP_SET=default \
    VERBOSE=1 --proxy-env=SOURCE_DATE_EPOCH FW_LTO="$opt" firmware_dfu fap_desktop_settings \
    > "$here/fixed-$mode.log" 2>&1
  d=build/f7-firmware-C
  t=toolchain/current/bin/arm-none-eabi
  "$t-gcc" --version | sed -n '1p'
  printf '%s:\n' "$mode"
  wc -c < "$d/firmware.bin"
  sha256sum "$d"/firmware.{bin,dfu} "$d"/.extapps/desktop_settings.fap
  out="$here/fixed-evidence/$mode"
  mkdir -p "$out"
  cp -p "$d"/firmware.{bin,dfu,elf} "$d"/toolbox/version.json "$d"/.extapps/desktop_settings.fap "$out/"
  "$t-objdump" -h "$d/firmware.elf" > "$out/sections.txt"
  "$t-nm" -a "$d/firmware.elf" > "$out/symbols.txt"
  sha256sum "$d/assets/compiled/firmware_api_table.h" targets/f7/api_symbols.csv > "$out/api.sha256"
  find "$d/lib" -maxdepth 1 -name '*.a' -print0 | sort -z | xargs -0 sha256sum > "$out/static-libraries.sha256"
  python3 "$here/archive_digest.py" "$d/lib" > "$out/static-libraries-no-ar-timestamps.sha256"
  mv build "build-$mode"
  mv .sconsign.dblite ".sconsign-$mode.dblite"
done
cmp build-{baseline,candidate}/f7-firmware-C/.extapps/desktop_settings.fap

cd -- "$here"
export F7_SRC="$repo" F7_GENERATED_BUILD="$repo/build-baseline"
export F7_NM="$repo/toolchain/current/bin/arm-none-eabi-nm"
python3 compare_commands.py --fixed
python3 compare_artifacts.py --fixed
python3 check_api_table.py --fixed
python3 check_startup.py --fixed
python3 check_hooks_source.py "$repo"
if grep -nE 'warning:.*(Wlto-type-mismatch|Wstringop-overread)' fixed-{baseline,candidate}.log; then
  exit 2
fi
```

</details>

<details>
<summary><code>locked-submodules.txt</code></summary>

```text
 df6334cdde0c88e22ab09ba16b12222cfccf834a applications/main/subghz_remote (df6334c)
 d827357e8bc3dc68d73158998e9fa52509f30836 assets/protobuf (0.29-1-gd827357)
 def7d2df2b0506d3d249334974f51e427c17a41c lib/FreeRTOS-Kernel (def7d2d)
 4273ca3211b99914f31518378fb590fbff064953 lib/FreeRTOS-Kernel/portable/ThirdParty/Community-Supported-Ports (4273ca3)
 3f9c99a682c5c796bb7eb89fd9c4385688fce27a lib/FreeRTOS-Kernel/portable/ThirdParty/Partner-Supported-Ports (3f9c99a)
 7398ccc91652a33483245200cfa1a83b073bc206 lib/heatshrink (heads/master)
 6ca2857519f996244f7b324dd227fdf0a075fffb lib/libusb_stm32 (6ca2857)
 107ea89daaefb9867ea9121002fbbdf926780e98 lib/mbedtls (107ea89)
 94599c0e3b5036e086446a51a3f79640f70f22f6 lib/mbedtls/framework (94599c0)
 1e921369b2c92bb219fcef84a37d4d2347794c0f lib/microtar (heads/master)
 62c8ac3e5d4a7a4f8757328e7a80286fde2686b6 lib/mlib (62c8ac3)
 6cfe48d6f1593f8fa5c0f90437f5e6522587745e lib/nanopb (6cfe48d)
 d1b860584dfe24d40d455ae624ed14600dfa93c9 lib/stm32wb_cmsis (d1b8605)
 133182d5583e998bb263cd947105be4df9c29cb3 lib/stm32wb_copro (v1.20.0)
 781464e628eb694b4c98a79ab524c293f527fc8d lib/stm32wb_copro/scripts (heads/dev)
 cfd0dd258cb031c95b2b2d6d04c19f9f625fe3e8 lib/stm32wb_hal (cfd0dd2)
```

</details>

<details>
<summary><code>prerequisites.patch</code></summary>

```diff
diff --git a/applications/services/expansion/expansion.c b/applications/services/expansion/expansion.c
index 3e8191f..8097a84 100644
--- a/applications/services/expansion/expansion.c
+++ b/applications/services/expansion/expansion.c
@@ -289,9 +289,7 @@ static void expansion_storage_callback(const void* message, void* context) {
     }
 }
 
-void expansion_on_system_start(void* arg) {
-    UNUSED(arg);
-
+void expansion_on_system_start(void) {
     Expansion* instance = expansion_alloc();
     furi_record_create(RECORD_EXPANSION, instance);
     furi_thread_start(instance->thread);
diff --git a/applications/services/gps/gps.c b/applications/services/gps/gps.c
index 12b7650..e3c0210 100644
--- a/applications/services/gps/gps.c
+++ b/applications/services/gps/gps.c
@@ -83,7 +83,6 @@ void gps_on_location(Gps* gps, GpsStatus status, const GpsLocation* location) {
     }
 }
 
-void gps_on_system_start(void* p) {
-    UNUSED(p);
+void gps_on_system_start(void) {
     furi_record_create(RECORD_GPS, gps_alloc());
 }
diff --git a/applications/services/locale/locale.c b/applications/services/locale/locale.c
index 6acf2ee..382e39b 100644
--- a/applications/services/locale/locale.c
+++ b/applications/services/locale/locale.c
@@ -106,7 +106,5 @@ void locale_format_date(
     }
 }
 
-int32_t locale_on_system_start(void* p) {
-    UNUSED(p);
-    return 0;
+void locale_on_system_start(void) {
 }
diff --git a/applications/services/namechanger/namechanger.c b/applications/services/namechanger/namechanger.c
index 6beef8d..a36d9a1 100644
--- a/applications/services/namechanger/namechanger.c
+++ b/applications/services/namechanger/namechanger.c
@@ -70,10 +70,9 @@ static bool namechanger_init() {
     return res;
 }
 
-int32_t namechanger_on_system_start(void* p) {
-    UNUSED(p);
+void namechanger_on_system_start(void) {
     if(furi_hal_rtc_get_boot_mode() != FuriHalRtcBootModeNormal) {
-        return 0;
+        return;
     }
 
     // Wait for all required services to start and create their records
@@ -82,7 +81,7 @@ int32_t namechanger_on_system_start(void* p) {
           !furi_record_exists(RECORD_STORAGE)) {
         timeout++;
         if(timeout > 250) {
-            return 0;
+            return;
         }
         furi_delay_ms(5);
     }
@@ -105,6 +104,4 @@ int32_t namechanger_on_system_start(void* p) {
         bt = NULL;
         furi_delay_ms(3);
     }
-
-    return 0;
 }
diff --git a/applications/services/network/network.c b/applications/services/network/network.c
index e86e5fc..7f5f129 100644
--- a/applications/services/network/network.c
+++ b/applications/services/network/network.c
@@ -159,7 +159,6 @@ void network_on_event(Network* network, const NetworkEvent* event) {
     }
 }
 
-void network_on_system_start(void* p) {
-    UNUSED(p);
+void network_on_system_start(void) {
     furi_record_create(RECORD_NETWORK, network_alloc());
 }
diff --git a/applications/services/rpc/rpc.c b/applications/services/rpc/rpc.c
index 7384805..70cbe35 100644
--- a/applications/services/rpc/rpc.c
+++ b/applications/services/rpc/rpc.c
@@ -438,8 +438,7 @@ void rpc_session_close(RpcSession* session) {
     furi_thread_flags_set(furi_thread_get_id(session->thread), RpcEvtDisconnect);
 }
 
-void rpc_on_system_start(void* p) {
-    UNUSED(p);
+void rpc_on_system_start(void) {
     Rpc* rpc = malloc(sizeof(Rpc));
 
     rpc->busy_mutex = furi_mutex_alloc(FuriMutexTypeNormal);
diff --git a/applications/settings/storage_settings/storage_settings.c b/applications/settings/storage_settings/storage_settings.c
index 570ca34..1dfdee9 100644
--- a/applications/settings/storage_settings/storage_settings.c
+++ b/applications/settings/storage_settings/storage_settings.c
@@ -1,17 +1,12 @@
 #include "storage_settings.h"
 
-const SubmenuSettingsHelperDescriptor descriptor_template = {
-    .app_name = "Storage",
-    .options_cnt = 6,
-    .options =
-        {
-            {.name = "About Internal Storage", .scene_id = StorageSettingsInternalInfo},
-            {.name = "About SD Card", .scene_id = StorageSettingsSDInfo},
-            {.name = "Unmount SD Card", .scene_id = StorageSettingsUnmountConfirm},
-            {.name = "Format SD Card", .scene_id = StorageSettingsFormatConfirm},
-            {.name = "Benchmark SD Card", .scene_id = StorageSettingsBenchmarkConfirm},
-            {.name = "Factory Reset", .scene_id = StorageSettingsFactoryReset},
-        },
+static const SubmenuSettingsHelperOption storage_settings_options[] = {
+    {.name = "About Internal Storage", .scene_id = StorageSettingsInternalInfo},
+    {.name = "About SD Card", .scene_id = StorageSettingsSDInfo},
+    {.name = "Unmount SD Card", .scene_id = StorageSettingsUnmountConfirm},
+    {.name = "Format SD Card", .scene_id = StorageSettingsFormatConfirm},
+    {.name = "Benchmark SD Card", .scene_id = StorageSettingsBenchmarkConfirm},
+    {.name = "Factory Reset", .scene_id = StorageSettingsFactoryReset},
 };
 
 static bool storage_settings_custom_event_callback(void* context, uint32_t event) {
@@ -54,11 +49,12 @@ static StorageSettings* storage_settings_alloc(void) {
     view_dispatcher_add_view(
         app->view_dispatcher, StorageSettingsViewDialogEx, dialog_ex_get_view(app->dialog_ex));
 
-    size_t descriptor_size =
-        sizeof(SubmenuSettingsHelperDescriptor) +
-        (descriptor_template.options_cnt * sizeof(SubmenuSettingsHelperOption));
-    app->helper_descriptor = malloc(descriptor_size);
-    memcpy(app->helper_descriptor, &descriptor_template, descriptor_size);
+    app->helper_descriptor =
+        malloc(sizeof(SubmenuSettingsHelperDescriptor) + sizeof(storage_settings_options));
+    app->helper_descriptor->app_name = "Storage";
+    app->helper_descriptor->options_cnt = COUNT_OF(storage_settings_options);
+    memcpy(
+        app->helper_descriptor->options, storage_settings_options, sizeof(storage_settings_options));
     app->settings_helper = submenu_settings_helpers_alloc(app->helper_descriptor);
     submenu_settings_helpers_assign_objects(
         app->settings_helper,
```

</details>

<details>
<summary><code>experimental-fw-lto.patch</code></summary>

```diff
--- a/site_scons/commandline.scons
+++ b/site_scons/commandline.scons
@@ -74,6 +74,11 @@
     BoolVariable(
         "LIB_DEBUG",
         help="Enable debug build for libraries",
+        default=False,
+    ),
+    BoolVariable(
+        "FW_LTO",
+        help="Opt in to LTO for the compact non-debug main firmware",
         default=False,
     ),
     BoolVariable(
--- a/firmware.scons
+++ b/firmware.scons
@@ -216,6 +216,16 @@
 # Debug
 # print(fwenv.Dump())
 
+# Apply LTO to built-in C/C++ code and the final main-firmware link only.
+# SDK, external applications, static libraries and the RAM updater remain unchanged.
+if (
+    fwenv["FW_LTO"]
+    and fwenv["IS_BASE_FIRMWARE"]
+    and fwenv["COMPACT"]
+    and not fwenv["DEBUG"]
+):
+    fwenv.AppendUnique(CCFLAGS=["-flto=1"], LINKFLAGS=["-flto=1"])
+
 # Full firmware definition
 fwelf = fwenv["FW_ELF"] = fwenv.Program(
     "${FIRMWARE_BUILD_CFG}",
```

</details>

<details>
<summary><code>archive_digest.py</code></summary>

```python
#!/usr/bin/env python3
"""Compare full ar archives after canonicalizing only the member timestamps."""
from hashlib import sha256
from pathlib import Path
import sys


def digest(path):
    data = bytearray(path.read_bytes())
    assert data[:8] == b'!<arch>\n', path
    pos = 8
    count = 0
    while pos < len(data):
        assert data[pos + 58:pos + 60] == b'`\n', (path,pos)
        size = int(data[pos+48:pos+58])
        timestamp = data[pos+16:pos+28].strip()
        assert (not timestamp or timestamp.isdigit()) and size >= 0, (path,pos)
        data[pos+16:pos+28] = b'0           '
        pos += 60 + size + (size & 1)
        count += 1
    assert pos == len(data), (path,pos,len(data))
    return sha256(data).hexdigest(), count


if __name__ == '__main__':
    archives = sorted(Path(sys.argv[1]).glob('*.a'))
    assert archives
    for archive in archives:
        value, count = digest(archive)
        print(f'{value}  {count:4d}  {archive.name}')
```

</details>

<details>
<summary><code>compare_commands.py</code></summary>

```python
#!/usr/bin/env python3
"""Ensure the candidate adds only the expected compile/link flag."""
from pathlib import Path
import shlex
import sys

LTO = '-flto=1'
BUILD = 'build/f7-firmware-C/'


def parse(path):
    compile_commands = {}
    for line in Path(path).read_text().splitlines():
        if not line.startswith(('arm-none-eabi-gcc -o ', 'arm-none-eabi-g++ -o ')):
            continue
        args = shlex.split(line)
        if not args[2].startswith(BUILD) or not args[2].endswith('.o') or '-c' not in args:
            continue
        assert args[2] not in compile_commands, args[2]
        compile_commands[args[2]] = args
    return compile_commands


def find_link(path):
    lines = Path(path).read_text().splitlines()
    links = []
    for number, line in enumerate(lines):
        if not line.startswith(('arm-none-eabi-gcc ', 'arm-none-eabi-g++ ')):
            continue
        args = shlex.split(line)
        if len(args) == 2 and args[1].startswith('@/tmp/'):
            for back in range(max(0, number - 4), number):
                if 'Using tempfile ' + args[1][1:] + ' for command line:' in lines[back]:
                    args = [args[0], *shlex.split(' '.join(lines[back+1:number]))]
                    break
        if len(args) > 2 and args[1:3] == ['-o', BUILD + 'firmware.elf']:
            links.append(args)
    assert len(links) == 1, (path, len(links))
    return links[0]


fixed = '--fixed' in sys.argv
prefix = 'fixed-' if fixed else ''
baseline = parse(prefix+'baseline.log')
candidate = parse(prefix+'candidate.log')
expected_compiles = {obj for obj in baseline if obj.startswith(BUILD + 'applications/')}
assert expected_compiles, 'No built-in application objects in baseline'
expected_set = baseline.keys() if fixed else expected_compiles
assert expected_set == candidate.keys(), {'missing': sorted(expected_set - candidate.keys()),
                                          'extra': sorted(candidate.keys() - expected_set)}
for obj, new in candidate.items():
    assert new.count(LTO) == (obj in expected_compiles), obj
    assert [arg for arg in new if arg != LTO] == baseline[obj], obj
assert all(LTO not in cmd for cmd in baseline.values())
print(f'Compile commands: {len(baseline)} baseline C/C++ objects; '
      f'{len(expected_compiles)} built-in C/C++ objects with only {LTO} added')

link_a = find_link(prefix+'baseline.log')
link_b = find_link(prefix+'candidate.log')
assert link_b.count(LTO) == 1 and LTO not in link_a
assert [arg for arg in link_b if arg != LTO] == link_a
print(f'Final linker commands: only {LTO} added to {len(link_a)} baseline arguments')

for (variant,commands) in [('baseline',baseline),('candidate',candidate)]:
    fap = [command for name, command in commands.items() if name.startswith(BUILD+'.extapps/')]
    assert bool(fap) == fixed, (variant,len(fap))
    assert all('-flto=1' not in command for command in fap)
    if fixed:
        print(f'{variant}: {len(fap)} unchanged external-app compile commands')
```

</details>

<details>
<summary><code>compare_artifacts.py</code></summary>

```python
#!/usr/bin/env python3
"""Cross-check equivalent firmware build metadata and validate flash payloads."""
import hashlib
import json
from pathlib import Path
import re
import struct
import sys
import zlib

FLASH_BASE = 0x08000000
PAGE_SIZE = 4096


def snapshot(path):
    binary = (path / 'firmware.bin').read_bytes()
    dfu = (path / 'firmware.dfu').read_bytes()
    assert struct.unpack_from('<5sBIB', dfu) == (b'DfuSe', 1, len(dfu) - 16, 1), path
    signature, alternate, named, name, target_size, count = struct.unpack_from('<6sBI255sII', dfu, 11)
    assert (signature, alternate, named, name.rstrip(b'\0'), target_size, count) == (
        b'Target', 0, 1, b'Flipper Zero F7', len(binary) + 8, 1
    ), path
    base, size = struct.unpack_from('<II', dfu, 285)
    assert (base, size) == (FLASH_BASE, len(binary)), path
    assert binary == dfu[293:-16], path
    assert dfu[-8:-4] == b'UFD\x10', path
    assert struct.unpack_from('<I', dfu, len(dfu) - 4)[0] == (~zlib.crc32(dfu[:-4]) & 0xFFFFFFFF), path
    section_lines = (path / 'sections.txt').read_text().splitlines()
    sections = {}
    flash = []
    for pos, line in enumerate(section_lines):
        match = re.match(r'^\s*\d+\s+(\S+)\s+([0-9a-f]{8})\s+([0-9a-f]{8})\s+([0-9a-f]{8})\b', line)
        if match is None:
            continue
        name, length, vma, lma = match.groups()
        length, vma, lma = (int(item, 16) for item in (length, vma, lma))
        sections[name] = length
        if 'LOAD' in section_lines[pos + 1] and FLASH_BASE <= lma < FLASH_BASE + 0x100000:
            flash.append((lma, lma + length))
    assert min(start for start, _ in flash) == FLASH_BASE, path
    assert max(end for _, end in flash) == (end := FLASH_BASE + size), path
    sp, reset_vector = struct.unpack_from('<II', binary)
    assert sp == 0x20030000 and reset_vector & 1 and base < reset_vector < end, path
    return {
        'bin_bytes': size,
        'dfu_bytes': len(dfu),
        'bin_sha256': hashlib.sha256(binary).hexdigest(),
        'dfu_sha256': hashlib.sha256(dfu).hexdigest(),
        'flash_first_address': f'0x{base:08X}',
        'flash_end_exclusive': f'0x{end:08X}',
        'flash_pages_spanned': (size + PAGE_SIZE - 1) // PAGE_SIZE,
        'reset_vector': f'0x{reset_vector:08X}',
        'sections_bytes': {name: sections[name] for name in ('.isr_vector', '.text', '.rodata', '.data', '.bss')},
    }


def main():
    baseline = Path('fixed-evidence' if '--fixed' in sys.argv else 'evidence') / 'baseline'
    candidate = baseline.parent / 'candidate'
    before = snapshot(baseline)
    after = snapshot(candidate)
    fixed = '--fixed' in sys.argv
    libraries = 'static-libraries-no-ar-timestamps.sha256' if fixed else 'static-libraries.sha256'
    for filename in ('version.json', 'api.sha256', libraries):
        assert (baseline / filename).read_bytes() == (candidate / filename).read_bytes(), filename
    symbols_before = (baseline / 'symbols.txt').read_text()
    symbols_after = (candidate / 'symbols.txt').read_text()
    for name in ('Reset_Handler', 'elf_resolve_from_hashtable'):
        assert re.search(r'\b' + name + r'(?:\.[^\s]+)?$', symbols_before, re.M), name
        assert re.search(r'\b' + name + r'(?:\.[^\s]+)?$', symbols_after, re.M), name
    savings = before['bin_bytes'] - after['bin_bytes']
    print(json.dumps({'baseline': before, 'candidate': after, 'savings_bytes': savings,
                      'savings_pct': round(100 * savings / before['bin_bytes'], 4),
                      'same_api_source_and_generated_table': True,
                      'same_version_json': True, 'same_static_library_archive_contents': True,
                      'static_archive_timestamp_normalization': fixed,
                      'dfu_savings_bytes': before['dfu_bytes'] - after['dfu_bytes']}, indent=2))


if __name__ == '__main__':
    main()
```

</details>

<details>
<summary><code>check_api_table.py</code></summary>

```python
#!/usr/bin/env python3
"""Check the API keys and address ranges in the linked F7 image, without device I/O."""
import csv
import os
from pathlib import Path
import re
import struct
import sys
import subprocess

FLASH_BASE = 0x08000000


def gnu_hash(name):
    value = 0x1505
    for byte in name.encode('ascii'):
        value = (value * 33 + byte) & 0xffffffff
    return value


def main():
    nm = os.environ.get('F7_NM', 'upstream/toolchain/current/bin/arm-none-eabi-nm')
    with open(Path(os.environ.get('F7_SRC', 'upstream')) / 'targets/f7/api_symbols.csv', newline='') as table:
        exports = {gnu_hash(row['name']): row['entry'] for row in csv.DictReader(table)
                   if row['status'] == '+' and row['entry'] in {'Function', 'Variable'}}
    for variant in ('baseline', 'candidate'):
        path = Path('fixed-evidence' if '--fixed' in sys.argv else 'evidence') / variant
        data = (path / 'firmware.bin').read_bytes()
        symbols = subprocess.check_output([nm, '-a', '-S', str(path / 'firmware.elf')], text=True)
        tbl = re.findall(r'^([0-9a-f]+) ([0-9a-f]+) [rR] \S*elf_api_table(?:\.[^\s]+)?$', symbols, re.M)
        assert len(tbl) == 1, (variant, tbl)
        addr, size = (int(part, 16) for part in tbl[0])
        allowed = []
        sections = (path / 'sections.txt').read_text().splitlines()
        for pos, line in enumerate(sections):
            row = re.match(r'^\s*\d+\s+(\S+)\s+([0-9a-f]{8})\s+([0-9a-f]{8})', line)
            if row and 'ALLOC' in sections[pos+1]:
                name, length, start = row.groups()
                start, length = int(start, 16), int(length, 16)
                if name != '.free_flash' and length:
                    allowed.append((start, start + length, 'CODE' in sections[pos+1]))
        assert size == 8 * len(exports) and 0 <= addr-FLASH_BASE <= len(data)-size, variant
        table = list(struct.iter_unpack('<II', data[addr-FLASH_BASE:addr-FLASH_BASE+size]))
        assert [key for key, _ in table] == sorted(exports), variant
        for key, pointer in table:
            if exports[key] == 'Function':
                assert pointer & 1 and any(a <= pointer - 1 < b and code for a,b,code in allowed), (variant, hex(key), hex(pointer))
            else:
                assert any(a <= pointer < b for a,b,_ in allowed), (variant, hex(key), hex(pointer))
        print(f'{variant}: {len(table)} sorted SDK API hashes matched; target address ranges verified')


if __name__ == '__main__':
    main()
```

</details>

<details>
<summary><code>check_startup.py</code></summary>

```python
#!/usr/bin/env python3
"""Verify linked startup table and Thumb entry addresses match unchanged manifest order."""
import argparse
import os
from pathlib import Path
import re
import struct
import subprocess

BASE = 0x08000000


def main():
    parser = argparse.ArgumentParser()
    parser.add_argument('--fixed', action='store_true')
    args = parser.parse_args()
    root = Path('fixed-evidence' if args.fixed else 'evidence')
    src = Path(os.environ.get('F7_GENERATED_BUILD', 'isolated/build-baseline' if args.fixed else 'upstream/build')) / 'f7-firmware-C/applications/applications.c'
    content = src.read_text()
    declarations = set(re.findall(r'^extern void (\w+)\(void\);$', content, re.M))
    block = content.split('const FlipperInternalOnStartHook FLIPPER_ON_SYSTEM_START[] = {\n', 1)[1].split('\n};', 1)[0]
    names = block.replace(',', '').split()
    assert len(names) == len(set(names)) and set(names) <= declarations
    t = os.environ.get('F7_NM', 'upstream/toolchain/current/bin/arm-none-eabi-nm')
    for variant in ('baseline','candidate'):
        image = (root/variant/'firmware.bin').read_bytes()
        output = subprocess.check_output([t,'-a','-S',str(root/variant/'firmware.elf')],text=True)
        output = [line.split() for line in output.splitlines()]
        symbols = {row[3]: (int(row[0],16),int(row[1],16)) for row in output if len(row) == 4}
        addr, size = symbols['FLIPPER_ON_SYSTEM_START']
        assert size == 4 * len(names), (addr,size)
        actual = struct.unpack_from('<'+'I'*len(names),image,addr-BASE)
        expected = tuple(symbols[name][0] | 1 for name in names)
        assert actual == expected, list(zip(names,actual,expected))
        if 'FLIPPER_ON_SYSTEM_START_COUNT' in symbols:
            assert struct.unpack_from('<I', image, symbols['FLIPPER_ON_SYSTEM_START_COUNT'][0]-BASE)[0] == len(names)
        print(f'{variant}: {len(names)} startup hooks preserved in order: {", ".join(names)}')


if __name__ == '__main__':
    main()
```

</details>

<details>
<summary><code>check_hooks_source.py</code></summary>

```python
#!/usr/bin/env python3
"""Compare the six fixed hooks with the pinned base after removing ABI-only edits."""
from pathlib import Path
import re
import subprocess
import sys

services=('rpc','gps','network','expansion','locale','namechanger')
repo=Path(sys.argv[1] if len(sys.argv)>1 else 'isolated')

for service in services:
    symbol=service+'_on_system_start'
    path=f'applications/services/{service}/{service}.c'
    before=subprocess.check_output(['git','-C',str(repo),'show','HEAD:'+path],text=True)
    after=(repo/path).read_text()
    def split(src):
        match = re.search(rf'^(void|int32_t) {symbol}\(([^)]*)\) {{\n(.*?)^}}',src,re.M|re.S)
        assert match is not None, symbol
        return match, src[:match.start()]+src[match.end():]
    old, rest_old=split(before)
    new, rest_new=split(after)
    assert rest_old==rest_new, f'{path}: unrelated changes'
    assert new[1]=='void' and new[2]=='void', symbol
    unused=re.fullmatch(r'void\* (\w+)',old[2]); assert unused, symbol
    old_body=re.sub(rf'^\s*UNUSED\({unused[1]}\);\s*', '', old[3], count=1)
    new_body=new[3]
    if old[1]=='int32_t':
        old_body=re.sub(r'\breturn 0;\s*\Z','',old_body)
        old_body=re.sub(r'\breturn 0;', 'return;', old_body)
    assert re.sub(r'\s','',old_body)==re.sub(r'\s','',new_body), f'{symbol}: behavior-related code changed'
    print(f'{symbol}: {old[1]}({old[2]}) -> void(void); identical statements/branches after ABI normalization')
```

</details>

## Вывод проверок и контрольные суммы

<details>
<summary>Чистый A/B: статическая сверка и SHA-256</summary>

```text
Compile commands: 830 baseline C/C++ objects; 227 built-in C/C++ objects with only -flto=1 added
Final linker commands: only -flto=1 added to 448 baseline arguments
baseline: 19 unchanged external-app compile commands
candidate: 19 unchanged external-app compile commands
{
  "baseline": {
    "bin_bytes": 875616,
    "dfu_bytes": 875925,
    "bin_sha256": "3937009322d4c54fd7007323dbbb81886b8ef3b3ac88051710f5bf6a5c76f8c7",
    "dfu_sha256": "49a246b01ca26fcd9c8238506d693d8c4b715d07dc7c278d69e525aab1199e8c",
    "flash_first_address": "0x08000000",
    "flash_end_exclusive": "0x080D5C60",
    "flash_pages_spanned": 214,
    "reset_vector": "0x08011859",
    "sections_bytes": {
      ".isr_vector": 316,
      ".text": 703188,
      ".rodata": 171392,
      ".data": 696,
      ".bss": 7544
    }
  },
  "candidate": {
    "bin_bytes": 868692,
    "dfu_bytes": 869001,
    "bin_sha256": "c8a04c5dc8389522718e5bb8d6683c667e7e1d891f1503db6ad9cc13324fe02a",
    "dfu_sha256": "23512716a7f3328ac544325edccda0bc9508713a8d9adefa227f8d328de931be",
    "flash_first_address": "0x08000000",
    "flash_end_exclusive": "0x080D4154",
    "flash_pages_spanned": 213,
    "reset_vector": "0x08011859",
    "sections_bytes": {
      ".isr_vector": 316,
      ".text": 696788,
      ".rodata": 170872,
      ".data": 692,
      ".bss": 7544
    }
  },
  "savings_bytes": 6924,
  "savings_pct": 0.7908,
  "same_api_source_and_generated_table": true,
  "same_version_json": true,
  "same_static_library_archive_contents": true,
  "static_archive_timestamp_normalization": true,
  "dfu_savings_bytes": 6924
}
baseline: 3046 sorted SDK API hashes matched; target address ranges verified
candidate: 3046 sorted SDK API hashes matched; target address ranges verified
baseline: 14 startup hooks preserved in order: cli_on_system_start, rpc_on_system_start, gps_on_system_start, network_on_system_start, power_on_system_start, storage_on_system_start, locale_on_system_start, loader_on_system_start, updater_on_system_start, expansion_on_system_start, subghz_dangerous_freq, clock_settings_start, findmy_startup, namechanger_on_system_start
candidate: 14 startup hooks preserved in order: cli_on_system_start, rpc_on_system_start, gps_on_system_start, network_on_system_start, power_on_system_start, storage_on_system_start, locale_on_system_start, loader_on_system_start, updater_on_system_start, expansion_on_system_start, subghz_dangerous_freq, clock_settings_start, findmy_startup, namechanger_on_system_start
rpc_on_system_start: void(void* p) -> void(void); identical statements/branches after ABI normalization
gps_on_system_start: void(void* p) -> void(void); identical statements/branches after ABI normalization
network_on_system_start: void(void* p) -> void(void); identical statements/branches after ABI normalization
expansion_on_system_start: void(void* arg) -> void(void); identical statements/branches after ABI normalization
locale_on_system_start: int32_t(void* p) -> void(void); identical statements/branches after ABI normalization
namechanger_on_system_start: int32_t(void* p) -> void(void); identical statements/branches after ABI normalization
c88ecb099357b1af36fa52917dfaecc86a450fb970044d33f191a60e96e5908f  fixed-evidence/baseline/desktop_settings.fap
c88ecb099357b1af36fa52917dfaecc86a450fb970044d33f191a60e96e5908f  fixed-evidence/candidate/desktop_settings.fap
Original LTO type and stringop warnings absent. Remaining diagnostics:
fixed-candidate.log:927:lto-wrapper: warning: using serial compilation of 7 LTRANS jobs
```

</details>

<details>
<summary>Изолированная проверка Storage на ARM</summary>

```text
original, LTO=off: 56-byte allocation, six labels and scene IDs, both SD labels: PASS
original, LTO=on: 56-byte allocation, six labels and scene IDs, both SD labels: PASS
fixed, LTO=off: 56-byte allocation, six labels and scene IDs, both SD labels: PASS
fixed, LTO=on: 56-byte allocation, six labels and scene IDs, both SD labels: PASS
```

</details>

<details>
<summary>Изолированные проверки varint и datetime на ARM</summary>

```text
varint: unchanged upstream assertions; test support adapted for emulation
  baseline: 4 cases, 800028 assertions, 0 failed
  candidate: 4 cases, 800028 assertions, 0 failed
datetime: unchanged upstream assertions; test support adapted for emulation
  baseline: 19 cases, 25 assertions, 0 failed
  candidate: 19 cases, 25 assertions, 0 failed
```

</details>
