## Контекст

Репозиторий: `td-json-client-prebuilt`  
Workflow: `.github/workflows/build-tdlib.yml`  

Цель: собрать Android `libtdjsonandroid.so` через GitHub Actions.

Подтверждающие фрагменты логов вынесены в отдельный файл:
`docs/android_ubuntu24_failure_logs.md`

## Базовая конфигурация, к которой вернулись

Чтобы зафиксировать исходную точку (“чистый старт”), `.github/workflows/build-tdlib.yml` приведён **в точности** к содержимому коммита `ce55df2` (историческая “рабочая” конфигурация, `runs-on: ubuntu-20.04`):

- `runs-on: ubuntu-20.04`
- NDK: `android-ndk-r20-linux-x86_64.zip`
- Парсинг `.github/workflows/td-version.json` через action `rgarcia-phi/json-to-variables@v1.1.0`
- Патч `td/example/build.sh` через `sed -i '5s/.../'` как было в исходном workflow

## Что ломалось при попытке перейти на `ubuntu-24.04` (фиксируем симптомы)

Ниже важна именно хронология: это не набор независимых ошибок, а цепочка попыток починить предыдущий сбой.

### 0) Ubuntu 20.04 runner: job не стартует (зависает в ожидании)

Коммит (база): `ce55df2` (`build-android.runs-on: ubuntu-20.04`).

Симптом из GitHub Actions:

- `Requested labels: ubuntu-20.04`
- `Waiting for a runner to pick up this job...`

Это и было причиной ухода на `ubuntu-24.04`: не потому что на `ubuntu-20.04` был подтверждён compile-failure, а потому что runner на `ubuntu-20.04` подбирался слишком долго, тогда как `ubuntu-24.04` стартовал сразу.

### 1) Первый подтверждённый compile-failure после перехода на `ubuntu-24.04`: CMake/TDLib падает на atomics

Коммит: `dd4d1c2`.

Что изменили относительно `ce55df2`:

- `build-android.runs-on`: `ubuntu-20.04` -> `ubuntu-24.04`
- `build-linux-x64.runs-on`: `ubuntu-20.04` -> `ubuntu-24.04`
- target по умолчанию для `workflow_dispatch`: `linux` -> `android`
- добавили системные пакеты `wget unzip zip curl ca-certificates`

NDK в этом коммите оставался старый: `android-ndk-r20-linux-x86_64.zip`.

Лог (симптом):

- `CMake Error ... FindAtomics.cmake:57 (message): Atomic operations library isn't found.`
- `-- Configuring incomplete, errors occurred!`

Это была первая подтверждённая ошибка сборки после перехода с `ubuntu-20.04` на `ubuntu-24.04`.

### 2) Попытка починить atomics через новый NDK: OpenSSL ломается на проверке NDK-layout

Замечено при конфиге: `runs-on: ubuntu-24.04` + NDK `r27d` (коммит `5c2cb03`).

Лог (симптом):

- `Run export ANDROID_NDK_HOME=/opt/android-ndk`
- `$ANDROID_NDK_HOME=/opt/android-ndk is invalid at (eval 10) line 37.`

Это происходило при использовании более новых NDK (например r27*), потому что OpenSSL 1.1.1g проверяет layout NDK и ожидает наличие `platforms/` или `AndroidVersion.txt`.

### 3) Попытка откатить NDK до legacy-layout: `r22b`, затем `r20`

Коммит `b3f4e3c`:

- заменили NDK `r27d` -> `r22b`, чтобы вернуть layout, который понимает OpenSSL 1.1.1g
- в `build.sh` добавили `-DANDROID_PLATFORM=android-21`

Коммит `67f5173`:

- заменили NDK `r22b` -> `r20`
- перестали менять только 5-ю строку `build.sh`, вместо этого начали патчить сам toolchain path и API level по ABI
- для `x86_64` и `arm64-v8a` выставляли API `21`, для остальных `16`

Оба коммита были попытками удержать совместимость OpenSSL со старым layout NDK и одновременно обойти Android/CMake-конфигурацию на `ubuntu-24.04`.

### 4) Попытка форсировать atomics-патч: сломался GitHub Action

Замечено при конфиге: `rgarcia-phi/json-to-variables@v1.2.1` (коммит `9ff22dc`).

Лог (симптом):

- `Error: Unable to resolve action rgarcia-phi/json-to-variables@v1.2.1, unable to find version v1.2.1`

Причина: у action нет тега/релиза `v1.2.1`.

Дополнительно в этом коммите пытались обойти atomics так:

- `actions/checkout@v4`
- `FORCE_JAVASCRIPT_ACTIONS_TO_NODE24: "true"`
- прямой патч `td/CMake/FindAtomics.cmake`, который сразу выставляет `ATOMICS_FOUND=TRUE` для Android

Но до проверки этой идеи job не доходил из-за несуществующей версии action.

### 5) Откат к базе и следующая попытка без битого action

Коммит `80f7ec1` (текущий `develop`) вернул workflow к конфигурации `ce55df2` и добавил этот документ.

При этом в `origin/develop` есть следующий коммит `84cf693`, который уже продолжает работу оттуда:

- снова `runs-on: ubuntu-24.04`
- чтение `.github/workflows/td-version.json` через `jq`, без `json-to-variables`
- `actions/checkout@v4`
- NDK снова `r20`
- atomics обходятся уже не патчем `FindAtomics.cmake`, а через параметры CMake в `build.sh`:
  `-DATOMICS_FOUND=TRUE -DCMAKE_TRY_COMPILE_TARGET_TYPE=STATIC_LIBRARY`

Это важная точка: если продолжать расследование дальше, логичнее стартовать не от текущего отката `80f7ec1`, а от `84cf693`, потому что он уже:

- не зависит от `ubuntu-20.04`
- не зависит от несуществующего action-тега
- оставляет совместимый с OpenSSL NDK `r20`
- изолирует актуальную проблему вокруг CMake/Android/atomics

## Дальше

Практическая стартовая точка для новой итерации:

- историческая база для сравнения: `ce55df2`
- бесполезный для продолжения откат: `80f7ec1`
- лучший следующий кандидат для продолжения на `ubuntu-24.04`: `84cf693`

То есть цепочка выглядит так:

- `ce55df2`: конфигурация на `ubuntu-20.04`, где проблема была в ожидании runner, а не в подтверждённой ошибке компиляции
- `dd4d1c2`: переход на `ubuntu-24.04`, первый подтверждённый compile-failure на atomics
- `5c2cb03`: попытка уйти на новый NDK, ломается OpenSSL
- `b3f4e3c`: OpenSSL-ошибка сохраняется
- `67f5173`: возврат к `NDK r20`, OpenSSL уходит, снова проявляется исходная проблема atomics
- `9ff22dc`: попытка форсировать atomics, но ломается action version
- `80f7ec1`: откат и фиксация симптомов
- `84cf693`: чистая продолженная попытка без битого action, от неё и стоит двигаться дальше
