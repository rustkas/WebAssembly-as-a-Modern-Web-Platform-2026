# Приложение B. Настройка окружения

Сборка проектов для WebAssembly требует определённого набора инструментов. Выбор конкретных утилит зависит от языка программирования и целевой среды выполнения — браузер, серверный рантайм или микроконтроллер. В этом приложении мы рассмотрим настройку трёх базовых инструментов: универсального набора `wasm-tools`, классического `wat2wasm` из WABT и `cargo-wasi` для Rust-разработки.

## wasm-tools

`wasm-tools` — это проект Bytecode Alliance, предоставляющий CLI-утилиты и Rust-библиотеки для низкоуровневой работы с WebAssembly-модулями . Это основной инструмент для работы с бинарным форматом WASM, валидации, парсинга и генерации кода.

### Установка

Для установки `wasm-tools` требуется Rust и Cargo . После установки Rust выполните:

```bash
cargo install --locked wasm-tools
```

Если у вас установлен `cargo binstall`, можно использовать его для установки предварительно собранных артефактов:

```bash
cargo binstall wasm-tools
```

### Проверка установки

```bash
wasm-tools --version
```

Для просмотра списка доступных подкоманд:

```bash
wasm-tools help
```

### Основные команды

| Команда | Назначение |
|---------|------------|
| `wasm-tools parse` | Парсинг бинарного WASM в текстовый формат WAT |
| `wasm-tools validate` | Валидация WASM-модуля |
| `wasm-tools print` | Вывод структуры модуля в читаемом виде |
| `wasm-tools component` | Работа с Component Model |
| `wasm-tools wit` | Работа с WIT-интерфейсами |

## wat2wasm

`wat2wasm` — это компилятор из текстового формата WebAssembly (WAT) в бинарный (WASM). Входит в состав WABT (WebAssembly Binary Toolkit) — набора инструментов, поддерживаемого командой WebAssembly . Инструмент полезен для ручного написания небольших WASM-модулей и для ознакомления с внутренним устройством языка.

### Установка

**macOS (через Homebrew):**
```bash
brew install wabt
```

**Windows (через Chocolatey):**
```bash
choco install wabt
```

**Linux (Debian/Ubuntu):**
```bash
sudo apt-get install wabt
```

Для других дистрибутивов Linux доступны сборки на GitHub-релизах WABT .

### Использование

Базовый синтаксис:

```bash
wat2wasm input.wat -o output.wasm
```

Пример с файлом `test.wat`:

```bash
wat2wasm test.wat -o test.wasm
```

Для детального вывода (с интерпретацией каждого байта) используется флаг `-v`:

```bash
wat2wasm test.wat -v
```

### Дополнительные команды WABT

Помимо `wat2wasm`, WABT включает:
- `wasm2wat` — обратное преобразование из бинарного в текстовый формат
- `wast2json` — преобразование тестовых файлов в JSON
- `wasm-validate` — валидация WASM-файлов
- `wasm-strip` — удаление отладочных секций

## cargo-wasi

`cargo-wasi` — это Cargo-субкоманда, которая упрощает сборку Rust-кода для WASI-целей . Она предоставляет удобные настройки по умолчанию для компиляции в `wasm32-wasi`, избавляя разработчика от необходимости вручную управлять множеством инструментов и флагов .

### Установка

Для установки требуется Rust . Установите подкоманду через Cargo:

```bash
cargo install cargo-wasi
```

Эта команда устанавливает предварительно собранный бинарный файл для большинства платформ . Если предварительно собранного бинарного файла нет, установка выполняется из исходного кода.

Если вы предпочитаете собирать из исходников:

```bash
cargo install cargo-wasi-src
```

### Проверка установки

```bash
cargo wasi --version
```

Команда должна вывести номер версии и информацию о ревизии Git, из которой был собран бинарный файл .

### Сборка проекта

Базовый пример сборки Rust-проекта для WASI:

```bash
cargo wasi build
```

Эта команда:
1. Собирает проект в режиме по умолчанию (debug) .
2. Компилирует для целевой платформы `wasm32-wasi`.
3. Использует оптимальные настройки для WASI-среды.

Для сборки в режиме релиза:

```bash
cargo wasi build --release
```

## Сборка проекта

Помимо специализированных WASM-инструментов, могут потребоваться дополнительные инструменты для полноценной работы.

### Для C/C++ (Emscripten)

Для компиляции C/C++ в WebAssembly используется Emscripten SDK . Пример минимальной сборки с помощью Clang:

```bash
clang fib.c -o fib.wasm \
  --target=wasm32 -nostdlib -fvisibility=default -fuse-ld=lld \
  -Xlinker --no-entry -Xlinker -export-dynamic
```

Этот пример создаёт WASM-модуль из C-кода без стандартной библиотеки, используя LLVM-линкер. Обратите внимание, что на macOS может потребоваться указать путь к Homebrew-версии LLVM: `$(brew --prefix llvm)/bin/clang` . На практике для серьёзных проектов на C/C++ чаще используется полноценный тулчейн Emscripten.

### Для Rust

Для компиляции Rust-кода для браузера (не через WASI) используется цель `wasm32-unknown-unknown`:

```bash
rustup target add wasm32-unknown-unknown
cargo build --target wasm32-unknown-unknown
```

Для сборки Rust-компонентов для Component Model используется цель `wasm32-wasip2`:

```bash
rustup target add wasm32-wasip2
```

### Для Go

Go поддерживает компиляцию в WebAssembly из коробки. Для браузерной цели:

```bash
GOOS=js GOARCH=wasm go build -o main.wasm main.go
```

Для WASI-цели (Go 1.21+):

```bash
GOOS=wasip1 GOARCH=wasm go build -o main.wasm main.go
```

### Для TypeScript/JavaScript

Экосистема JavaScript использует инструменты JCO для работы с WASM-компонентами . Для установки TypeScript:

```bash
npm install -g typescript
```

### wasmCloud и wash

wasmCloud предоставляет CLI `wash` для работы с WASM-компонентами в серверной среде :

**macOS/Linux (через скрипт):**
```bash
curl -fsSL https://wasmcloud.com/sh | bash
```

**macOS (через Homebrew):**
```bash
brew install wasmcloud/wasmcloud/wash
```

**Windows (через winget):**
```bash
winget install wasmCloud.wash
```

Проверка установки:

```bash
wash -V
```

### Проверка работоспособности

После установки инструментов рекомендуется проверить окружение. Для Rust/WASI:

```bash
cargo wasi --version
```

Для проверки WASM-тулчейна целиком:

```bash
wasm-tools --version
wat2wasm --version
```

Если все команды выводят версии, окружение настроено корректно.