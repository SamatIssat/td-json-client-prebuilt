## Контекст

Репозиторий: `td-json-client-prebuilt`  
Workflow: `.github/workflows/build-tdlib.yml`  

Цель: собрать Android `libtdjsonandroid.so` через GitHub Actions.

## Базовая конфигурация, к которой вернулись

Чтобы зафиксировать исходную точку (“чистый старт”), `.github/workflows/build-tdlib.yml` приведён **в точности** к содержимому коммита `ce55df2` (историческая “рабочая” конфигурация, `runs-on: ubuntu-20.04`):

- `runs-on: ubuntu-20.04`
- NDK: `android-ndk-r20-linux-x86_64.zip`
- Парсинг `.github/workflows/td-version.json` через action `rgarcia-phi/json-to-variables@v1.1.0`
- Патч `td/example/build.sh` через `sed -i '5s/.../'` как было в исходном workflow

## Что ломалось при попытке перейти на `ubuntu-24.04` (фиксируем симптомы)

### 0) Ubuntu 20.04 runner: job не стартует (зависает в ожидании)

Коммит (база): `ce55df2` (`build-android.runs-on: ubuntu-20.04`).

Симптом из GitHub Actions:

- `Requested labels: ubuntu-20.04`
- `Waiting for a runner to pick up this job...`

Это означает, что job не получил доступный runner с нужным образом (или попал в очередь/ограничения).

### 1) OpenSSL: `$ANDROID_NDK_HOME=... is invalid`

Замечено при конфиге: `runs-on: ubuntu-24.04` + NDK `r27d` (коммит `5c2cb03`).

Лог (симптом):

- `Run export ANDROID_NDK_HOME=/opt/android-ndk`
- `$ANDROID_NDK_HOME=/opt/android-ndk is invalid at (eval 10) line 37.`

Это происходило при использовании более новых NDK (например r27*), потому что OpenSSL 1.1.1g проверяет layout NDK и ожидает наличие `platforms/` или `AndroidVersion.txt`.

### 2) TDLib/CMake: `Atomic operations library isn't found`

Замечено при конфиге: `runs-on: ubuntu-24.04` + NDK `r20` (коммит `dd4d1c2`).

Лог (симптом):

- `CMake Error ... FindAtomics.cmake:57 (message): Atomic operations library isn't found.`
- `-- Configuring incomplete, errors occurred!`

Это падение происходило на этапе конфигурации CMake при кросс-компиляции Android.

### 3) Action version: `unable to find version v1.2.1`

Замечено при конфиге: `rgarcia-phi/json-to-variables@v1.2.1` (коммит `9ff22dc`).

Лог (симптом):

- `Error: Unable to resolve action rgarcia-phi/json-to-variables@v1.2.1, unable to find version v1.2.1`

Причина: у action нет тега/релиза `v1.2.1`.

## Дальше

Следующий шаг — выбрать стратегию для `ubuntu-24.04` (NDK/openssl/atomics), но сначала фиксируем эту страницу как “что именно ломалось” и имеем базовую точку (ubuntu-20.04) для сравнения.
