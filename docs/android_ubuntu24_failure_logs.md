## Android Build Failure Logs

Источник: логи GitHub Actions, предоставленные пользователем в этом расследовании.

Цель файла: хранить короткие подтверждающие фрагменты вокруг фатальных ошибок, чтобы можно было быстро сверить summary из `docs/android_ubuntu24_failures.md` с реальными симптомами.

## Базовая интерпретация

- `ubuntu-20.04`: подтверждённой compile-ошибки нет; подтверждена проблема с долгим ожиданием runner
- `dd4d1c2`: первая подтверждённая compile-ошибка на `ubuntu-24.04` это `Atomic operations library isn't found`
- `5c2cb03`: после смены NDK появилась OpenSSL-ошибка `$ANDROID_NDK_HOME=/opt/android-ndk is invalid`
- `b3f4e3c`: OpenSSL-ошибка сохранилась
- `67f5173`: после возврата к `NDK r20` OpenSSL-ошибка ушла, снова проявилась ошибка `Atomic operations library isn't found`

## ce55df2

Статус:

- `build-android.runs-on: ubuntu-20.04`
- compile-failure не подтверждён логом
- подтверждён только симптом ожидания runner

Симптом:

```text
Requested labels: ubuntu-20.04
Waiting for a runner to pick up this job...
```

Вывод:

- переход на `ubuntu-24.04` был сделан из-за доступности runner, а не из-за подтверждённой ошибки компиляции на `ubuntu-20.04`

## dd4d1c2

Статус:

- первый подтверждённый compile-failure после перехода на `ubuntu-24.04`
- NDK ещё `r20`
- ошибка в `TDLib/CMake/FindAtomics.cmake`

Фатальный фрагмент:

```text
-- Git state: 0da5c72f8365fb4857096e716d53175ddbdf5a15
-- Found ZLIB: /opt/android-ndk/toolchains/llvm/prebuilt/linux-x86_64/sysroot/usr/lib/aarch64-linux-android/libz.a (found version "1.2.7")
-- Found ZLIB: /opt/android-ndk/toolchains/llvm/prebuilt/linux-x86_64/sysroot/usr/include /opt/android-ndk/toolchains/llvm/prebuilt/linux-x86_64/sysroot/usr/lib/aarch64-linux-android/libz.a
-- Performing Test ATOMICS_FOUND
-- Performing Test ATOMICS_FOUND - Failed
-- Performing Test ATOMICS_FOUND
-- Performing Test ATOMICS_FOUND - Failed
CMake Error at /home/runner/work/td-json-client-prebuilt/td-json-client-prebuilt/td/CMake/FindAtomics.cmake:57 (message):
  Atomic operations library isn't found.
Call Stack (most recent call first):
  /home/runner/work/td-json-client-prebuilt/td-json-client-prebuilt/td/tdutils/CMakeLists.txt:409 (find_package)

-- Configuring incomplete, errors occurred!
Error: Process completed with exit code 1.
```

Вывод:

- первая подтверждённая ошибка сборки после перехода на `ubuntu-24.04` это именно `atomics`

## 5c2cb03

Статус:

- попытка уйти на новый NDK для `ubuntu-24.04`
- ошибка уже не в `atomics`, а на этапе OpenSSL

Фатальный фрагмент:

```text
Run export ANDROID_NDK_HOME=/opt/android-ndk
$ANDROID_NDK_HOME=/opt/android-ndk is invalid at (eval 10) line 37.
Configuring OpenSSL version 1.1.1g (0x1010107fL) for android-x86_64
Using os-specific seed configuration
$ANDROID_NDK_HOME=/opt/android-ndk is invalid at (eval 10) line 37.
Configuring OpenSSL version 1.1.1g (0x1010107fL) for android-arm
Using os-specific seed configuration
$ANDROID_NDK_HOME=/opt/android-ndk is invalid at (eval 10) line 37.
Configuring OpenSSL version 1.1.1g (0x1010107fL) for android-x86
Using os-specific seed configuration
$ANDROID_NDK_HOME=/opt/android-ndk is invalid at (eval 10) line 37.
Configuring OpenSSL version 1.1.1g (0x1010107fL) for android-arm64
Using os-specific seed configuration
Error: Process completed with exit code 1.
```

Вывод:

- смена NDK увела сборку в отдельный тупик совместимости OpenSSL 1.1.1g с layout NDK

## b3f4e3c

Статус:

- попытка вернуть legacy-layout NDK
- OpenSSL-ошибка всё ещё сохраняется

Фатальный фрагмент:

```text
Run export ANDROID_NDK_HOME=/opt/android-ndk
$ANDROID_NDK_HOME=/opt/android-ndk is invalid at (eval 10) line 37.
Configuring OpenSSL version 1.1.1g (0x1010107fL) for android-x86_64
Using os-specific seed configuration
$ANDROID_NDK_HOME=/opt/android-ndk is invalid at (eval 10) line 37.
Configuring OpenSSL version 1.1.1g (0x1010107fL) for android-arm
Using os-specific seed configuration
$ANDROID_NDK_HOME=/opt/android-ndk is invalid at (eval 10) line 37.
Configuring OpenSSL version 1.1.1g (0x1010107fL) for android-x86
Using os-specific seed configuration
$ANDROID_NDK_HOME=/opt/android-ndk is invalid at (eval 10) line 37.
Configuring OpenSSL version 1.1.1g (0x1010107fL) for android-arm64
Using os-specific seed configuration
Error: Process completed with exit code 1.
```

Вывод:

- попытка с `r22b` ещё не возвращает сборку к исходной проблеме `atomics`, потому что OpenSSL падает раньше

## 67f5173

Статус:

- возврат к `NDK r20`
- OpenSSL уже не блокирует пайплайн
- снова воспроизводится исходная ошибка `atomics`

Фатальный фрагмент:

```text
-- Git state: 0da5c72f8365fb4857096e716d53175ddbdf5a15
-- Found ZLIB: /opt/android-ndk/toolchains/llvm/prebuilt/linux-x86_64/sysroot/usr/lib/aarch64-linux-android/libz.a (found version "1.2.7")
-- Found ZLIB: /opt/android-ndk/toolchains/llvm/prebuilt/linux-x86_64/sysroot/usr/include /opt/android-ndk/toolchains/llvm/prebuilt/linux-x86_64/sysroot/usr/lib/aarch64-linux-android/libz.a
-- Performing Test ATOMICS_FOUND
-- Performing Test ATOMICS_FOUND - Failed
-- Performing Test ATOMICS_FOUND
-- Performing Test ATOMICS_FOUND - Failed
CMake Error at /home/runner/work/td-json-client-prebuilt/td-json-client-prebuilt/td/CMake/FindAtomics.cmake:57 (message):
  Atomic operations library isn't found.
Call Stack (most recent call first):
  /home/runner/work/td-json-client-prebuilt/td-json-client-prebuilt/td/tdutils/CMakeLists.txt:409 (find_package)

-- Configuring incomplete, errors occurred!
Error: Process completed with exit code 1.
```

Вывод:

- после возврата к совместимому с OpenSSL NDK снова видна настоящая активная проблема: `FindAtomics.cmake`

## Замечания

- `CMake Deprecation Warning at /opt/android-ndk/build/cmake/android.toolchain.cmake:38` не является фатальной причиной падения в этих логах
- многочисленные `HAVE_CXX_FLAG_* - Failed` сами по себе не являются ключевой ошибкой
- фатальные ошибки в подтверждённых логах здесь только двух типов:
  - `Atomic operations library isn't found`
  - `$ANDROID_NDK_HOME=/opt/android-ndk is invalid`
