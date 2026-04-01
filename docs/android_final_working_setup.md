## Android Final Working Setup

Этот документ описывает текущую рабочую конфигурацию сборки Android `libtdjsonandroid.so` в GitHub Actions.

Подробная история с промежуточными ошибками:

- `docs/android_ubuntu24_failures.md`
- `docs/android_ubuntu24_failure_logs.md`

## Как запускается сборка

- workflow запускается только вручную через `workflow_dispatch`
- автоматический запуск по `push` отключён
- для Android нужно выбирать `target=android`

## Рабочая конфигурация

- runner: `ubuntu-24.04`
- Android NDK: `r20`
- сборка OpenSSL 1.1.1g проходит на `NDK r20`
- сборка Android выполняется через patched `td/example/build.sh`

## Что важно в рабочем решении

### 1) Runner

Используется `ubuntu-24.04`, потому что `ubuntu-20.04` слишком долго ждал runner.

### 2) NDK

Используется именно `android-ndk-r20-linux-x86_64.zip`.

Причина:

- с более новыми NDK возникала ошибка OpenSSL вида
  `$ANDROID_NDK_HOME=/opt/android-ndk is invalid`
- `r20` оказался совместим с текущей схемой сборки OpenSSL 1.1.1g

### 3) TDLib atomics check

В `td/CMake/FindAtomics.cmake` для Android принудительно выставляется успешная проверка atomics.

Причина:

- на `ubuntu-24.04` Android cross-compile падал на
  `Atomic operations library isn't found`
- это обходится прямым Android-specific патчем `FindAtomics.cmake`

### 4) Linker flag

Из CMake-файлов TDLib удаляется флаг:

- `-Wl,--icf=safe`

Причина:

- линкер из `NDK r20` не понимает этот флаг
- без удаления сборка падала на
  `ld: unrecognized option '--icf=safe'`

### 5) build.sh patch

Перед сборкой патчится `td/example/build.sh`:

- toolchain path переводится на `/opt/android-ndk/build/cmake/android.toolchain.cmake`
- API level выставляется по ABI
- для `x86_64` и `arm64-v8a` используется `android-21`
- для остальных ABI используется `android-16`

## Что не менять без причины

- не менять `NDK r20` на более новый без отдельной проверки OpenSSL
- не убирать Android patch для `FindAtomics.cmake`, пока не доказано, что сборка проходит без него
- не возвращать `-Wl,--icf=safe` для `NDK r20`
- не включать автозапуск по `push`, если Android по-прежнему предполагается запускать вручную

## Короткий вывод

Финальная рабочая схема для Android сейчас такая:

- manual only
- `ubuntu-24.04`
- `NDK r20`
- patch `FindAtomics.cmake` для Android
- удалить `-Wl,--icf=safe`
- patched `build.sh` с правильным toolchain path и API level
