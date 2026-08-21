# Глава 14. Rust → WebAssembly

Rust и WebAssembly образуют одну из самых сильных связок в современной веб-разработке. Rust предоставляет безопасность памяти, предсказуемую производительность и минимальный рантайм, что делает его идеальным кандидатом для компиляции в WebAssembly . WebAssembly, в свою очередь, даёт Rust-коду возможность выполняться в браузере с производительностью, близкой к нативной.

В этой главе мы рассмотрим, как Rust-код компилируется в WebAssembly, как он взаимодействует с JavaScript и DOM, как управляется память и обрабатываются ошибки, и как эта связка используется в production-средах.

## wasm-bindgen и стратегии работы с DOM

Основным инструментом для связывания Rust и JavaScript является **wasm-bindgen** — проект, генерирующий привязки между Rust-кодом и JavaScript-окружением .

### Макрос `#[wasm_bindgen]`

Центральный элемент wasm-bindgen — макрос `#[wasm_bindgen]`, который позволяет экспортировать Rust-функции в JavaScript и импортировать JavaScript-функции в Rust .

**Экспорт Rust-функции в JavaScript:**
```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub fn greet(name: &str) -> String {
    format!("Hello, {}!", name)
}
```

**Импорт JavaScript-функции в Rust:**
```rust
#[wasm_bindgen]
extern "C" {
    #[wasm_bindgen(js_namespace = console)]
    fn log(s: &str);

    #[wasm_bindgen(js_namespace = performance)]
    fn now() -> f64;
}
```

### Точка входа

Макрос `#[wasm_bindgen(start)]` отмечает функцию, которая вызывается автоматически при загрузке WASM-модуля. Это аналог функции `main` в браузерном контексте :

```rust
#[wasm_bindgen(start)]
pub fn main() -> Result<(), JsValue> {
    web_sys::console::log_1(&"Module loaded".into());
    Ok(())
}
```

### Стратегии работы с DOM

Rust не имеет прямого доступа к DOM. Взаимодействие с DOM осуществляется через JavaScript-прокси, генерируемые wasm-bindgen. Существует три основных подхода.

**1. Прямое использование web-sys**

Библиотека **tairitsu-web** (часть full-stack фреймворка Tairitsu на основе WASM Component Model) предлагает несколько реализаций абстрактного трейта `Platform`, определённого в `tairitsu-vdom`: `WitPlatform` — типобезопасная реализация через WIT (WebAssembly Interface Types) для окружений `wasm32`, использующая паттерн непрозрачного дескриптора (`u64`); `BrowserPlatform` — прямая интеграция через стандартные привязки `web-sys`; и `SsrPlatform` — серверная цель, сериализующая деревья виртуального DOM в HTML-строки для server-side rendering. WIT-подход (`WitPlatform`) использует непрозрачные дескрипторы для ссылок на DOM-элементы и пакетные операции для группировки обновлений, что снижает количество вызовов на границе.:

```rust
use web_sys::{CanvasRenderingContext2d, HtmlCanvasElement};

#[wasm_bindgen]
pub fn draw(canvas: &HtmlCanvasElement, width: u32, height: u32) {
    canvas.set_width(width);
    canvas.set_height(height);

    let ctx = canvas
        .get_context("2d")
        .unwrap()
        .unwrap()
        .dyn_into::<CanvasRenderingContext2d>()
        .unwrap();

    ctx.set_fill_style(&"#ff6600".into());
    ctx.fill_rect(10.0, 10.0, 100.0, 100.0);
}
```

**2. Компонентная модель через WIT**

В 2026 году набирает популярность подход, использующий Component Model и WIT-интерфейсы. Вместо прямых вызовов web-sys, DOM-операции описываются в WIT и выполняются через компонентную модель. Это даёт лучшую типобезопасность и позволяет компоновать DOM-логику из разных языков .

Библиотека **tairitsu-web** предлагает две реализации: `WitPlatform` (через Component Model) и `BrowserPlatform` (через web-sys). WIT-подход использует непрозрачные дескрипторы (`u64`) для ссылок на DOM-элементы и пакетные операции для группировки обновлений, что снижает количество вызовов на границе .

**3. Высокоуровневые UI-фреймворки**

Существуют фреймворки, полностью реализованные на Rust и компилируемые в WASM, которые скрывают работу с DOM за декларативным API. Например, фреймворк **EUV** предоставляет виртуальный DOM, реактивные сигналы и HTML-макросы для создания UI на Rust, компилируемого в WebAssembly .

```rust
// Пример UI-фреймворка на Rust (EUV)
fn render() -> VNode {
    div {
        class: "container",
        h1 { "Hello, World!" },
        button {
            onclick: |_| { /* реактивная логика */ },
            "Click me"
        }
    }
}
```

### Стоимость пересечения границы

Каждый вызов между Rust и JavaScript имеет накладные расходы :

| Операция | Относительная стоимость |
|----------|------------------------|
| Rust → Rust | 1× |
| JS → Rust (примитивные аргументы) | ~10× |
| JS → Rust (копирование строк/массивов) | ~50× |
| JS → Rust (сериализация через serde) | ~100× |
| Rust → DOM (через web-sys) | ~200× |

**Ключевое правило оптимизации**: группировать работу на стороне Rust и минимизировать количество переходов между Rust и JavaScript. Не пересекать границу в плотных циклах.

## Отсутствие сборщика мусора

Одно из ключевых преимуществ Rust в контексте WebAssembly — отсутствие сборщика мусора. Rust управляет памятью через модель владения (ownership) и заимствования (borrowing): у каждого значения ровно один владелец, а компилятор с помощью системы времени жизни (lifetimes) статически проверяет, что ссылки не переживают данные, на которые указывают. Подсчёт ссылок (`Rc`/`Arc`) — это не основной механизм, а отдельный, опциональный инструмент для тех редких случаев, когда данными нужно явно владеть совместно; в большинстве Rust-кода, компилируемого в WASM, он не используется вовсе.


В отличие от языков с GC, Rust-код в WebAssembly не создаёт непредсказуемых пауз и не требует встраивания сборщика в бинарный файл. Это приводит к нескольким важным следствиям :

**Меньший размер бинарных файлов.** Бинарник не содержит GC-рантайма. Hello World на Rust для WASM занимает около 1.8 КБ .

**Предсказуемая производительность.** Время выполнения не зависит от цикла сборки мусора.

**Ручное управление памятью.** Разработчик контролирует, когда и как выделяется и освобождается память. Это особенно важно для приложений с жёсткими требованиями к задержке.

### Модель памяти на границе

При передаче данных между Rust и JavaScript важно понимать, кому принадлежит память :

```rust
// Rust владеет памятью в линейной памяти WASM
#[wasm_bindgen]
pub fn get_buffer() -> Vec<u8> {
    vec![0u8; 1024] // выделено в WASM-куче
}

// JavaScript получает *копию* данных (если не использовать указатели)
// Чтобы избежать копирования, возвращаем указатель + длину:
#[wasm_bindgen]
pub fn get_ptr() -> *const u8 {
    let buf = vec![0u8; 1024];
    let ptr = buf.as_ptr();
    std::mem::forget(buf); // предотвращаем освобождение!
    ptr
}
```

В примере выше `std::mem::forget(buf)` предотвращает освобождение памяти. Если этого не сделать, указатель станет недействительным сразу после возврата из функции.

## web-sys и js-sys

Библиотеки **web-sys** и **js-sys** — основа работы с браузерными API из Rust.

### js-sys

`js-sys` предоставляет привязки к стандартному JavaScript API — объектам `Object`, `Array`, `String`, `Promise`, `Date` и другим глобальным конструкторам и функциям . Он также содержит привязки к ECMAScript API, таким как `ArrayBuffer`, `DataView`, `Map`, `Set` .

```rust
use js_sys::{Array, Object, Reflect};

let obj = Object::new();
Reflect::set(&obj, &"key".into(), &"value".into()).unwrap();

let arr = Array::new();
arr.push(&1);
arr.push(&2);
```

### web-sys

`web-sys` содержит привязки ко всем Web API — DOM, Canvas, WebGL, WebSocket, IndexedDB, WebGPU и другим. Весь код web-sys генерируется из WebIDL-спецификаций и использует реализацию макроса `#[js_sys]` для генерации привязок .

 Актуальные версии на середину 2026 года:
 - `js-sys`: 0.3.95
 - `web-sys`: 0.3.95
 - `wasm-bindgen`: 0.2.118 (вышел 10 апреля 2026 года)

### Типы и преобразования

wasm-bindgen автоматически преобразует типы между Rust и JavaScript :

| Rust Type | JavaScript Type | Направление |
|-----------|-----------------|-------------|
| `i32` / `u32` | `number` | Оба |
| `f64` | `number` | Оба |
| `bool` | `boolean` | Оба |
| `String` | `string` | Оба |
| `&str` | `string` | Rust ← JS |
| `Vec<T>` | `Array` / `TypedArray` | Rust ← JS |
| `JsValue` | `any` | Оба |
| `*const T` | `number` (указатель) | Rust → JS |

Для передачи сложных структур используется сериализация через `serde-wasm-bindgen` :

```rust
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize)]
pub struct Metrics {
    pub sum: i64,
    pub mean: f64,
    pub count: usize,
}

#[wasm_bindgen]
pub fn analyze(data: &[i32]) -> JsValue {
    let sum: i64 = data.iter().map(|&x| x as i64).sum();
    let mean = sum as f64 / data.len() as f64;
    let result = Metrics { sum, mean, count: data.len() };
    serde_wasm_bindgen::to_value(&result).unwrap()
}
```

### Обработка JsValue

Тип `JsValue` — это opaque-тип, представляющий любое JavaScript-значение. Он может быть использован для передачи произвольных данных между Rust и JavaScript без преобразования .

```rust
#[wasm_bindgen]
pub fn process(data: JsValue) -> JsValue {
    // data может быть чем угодно: объектом, массивом, числом
    // Возвращаем то же значение или трансформируем
    data
}
```

Важно: `JsValue` не является типом, с которым можно работать напрямую в Rust (нельзя вызвать методы или прочитать поля). Для этого требуется преобразование в конкретный тип (`js_sys::Object`, `web_sys::Element`) или сериализация.

## Паника и обработка ошибок

Обработка ошибок в Rust-коде, компилируемом в WebAssembly, имеет особенности, связанные с моделью выполнения WASM и интеграцией с JavaScript.

### WebAssembly Exception Handling

Основой для обработки паник и исключений в Rust/WASM является **WebAssembly Exception Handling** — спецификация, получившая широкую поддержку в 2023 году .

В Rust по умолчанию для целевой платформы `wasm32-unknown-unknown` используется `panic=abort` — любая паника приводит к немедленной ловушке (trap) и завершению WASM-модуля . Это означает, что при панике WASM-инстанс становится невалидным и не может использоваться для последующих вызовов.

### panic=unwind

С 2025-2026 годов стало возможным использовать `panic=unwind` для WebAssembly . Это позволяет:

- Выполнять деструкторы при панике;
- Восстанавливаться после паники без перезагрузки WASM-инстанса;
- Сохранять состояние приложения (например, в Durable Objects).

Для включения необходимы :
1. Сборка с флагом `-Cpanic=unwind`;
2. Использование экспериментального ночного Rust-тулчейна;
3. Сборка стандартной библиотеки (`-Zbuild-std`);
4. Поддержка современной версии WebAssembly Exception Handling в рантайме (вместо устаревшей "legacy" версии) .

```bash
RUSTFLAGS='-Cpanic=unwind' cargo +nightly build -Z build-std --target wasm32-unknown-unknown
```

Для WASI-целей поддержка `-Cpanic=unwind` также доступна на ночных сборках с `wasm32-wasip2` :

```bash
RUSTFLAGS='-Cpanic=unwind' cargo +nightly run -Z build-std --target wasm32-wasip2
```

Ключевое изменение — использование `extern "C-unwind"` вместо `extern "C"` для экспортированных функций, чтобы позволить раскрутке стека пересекать границу Rust↔JS .

### Перехват паник на границе Rust-JS

Cloudflare, активно использующий Rust Workers, реализовал механизм перехвата паник на границе Rust↔JS :

- **panic=unwind**: паники в экспортированных Rust-функциях перехватываются wasm-bindgen и передаются в JavaScript как исключения `PanicError`. Асинхронные экспорты отклоняют Promise с `PanicError`. Деструкторы Rust выполняются корректно .

- **Безопасность раскрутки**: для замыканий, которые не являются unwind-safe, добавлен `Closure::new_aborting`, завершающий выполнение при панике вместо раскрутки .

### Обработка аварийных завершений (aborts)

Некоторые ошибки не могут быть раскручены: нехватка памяти, нарушения безопасности, необработанные паники в неустойчивом к раскрутке коде. Для таких случаев wasm-bindgen предоставляет механизм восстановления после abort :

- **set_on_abort** — хук, вызываемый при abort, позволяющий выполнить восстановление.
- **reset-state-function** — экспериментальная возможность, позволяющая сбросить WASM-инстанс в начальное состояние без перезагрузки модуля .

```rust
// Установка обработчика abort
wasm_bindgen::set_on_abort(|| {
    // Восстановление состояния
    reinitialize_state();
});
```

При abort существующие экземпляры классов становятся невалидными, но новые могут быть созданы .

### Отлов ошибок в production

В 2026 году стандартной практикой стала установка глобального перехватчика паник :

```rust
use std::panic;
use wasm_bindgen::prelude::*;

#[derive(serde::Serialize)]
struct ErrorPayload {
    msg_type: &'static str,
    source: &'static str,
    message: String,
    location: String,
}

pub fn install_panic_hook() {
    panic::set_hook(Box::new(|info| {
        let message = if let Some(s) = info.payload().downcast_ref::<&str>() {
            s.to_string()
        } else if let Some(s) = info.payload().downcast_ref::<String>() {
            s.clone()
        } else {
            "Unknown Rust Panic".to_string()
        };

        let location = info.location()
            .map(|loc| format!("{}:{}:{}", loc.file(), loc.line(), loc.column()))
            .unwrap_or_else(|| "unknown_location".to_string());

        // Отправляем ошибку в JavaScript через postMessage
        let payload = ErrorPayload {
            msg_type: "ENGINE_ERROR",
            source: "RUST_PANIC",
            message,
            location,
        };
        send_error_to_js(&payload);
    }));
}
```

Этот подход используется в production-приложениях, где критически важно получать информацию об ошибках в UI .

### Интеграция с отладкой

Отладка Rust/WASM-приложений в 2026 году включает несколько уровней :

1. **Source maps** (DWARF) для привязки WASM-инструкций к исходному Rust-коду.
2. **Секция имён** в WASM-модуле для читаемых имён функций и переменных.
3. **Пользовательские кастомные секции** для метаданных отладки.
4. **Глобальный перехватчик паник** для передачи ошибок в UI.

---

Связка Rust и WebAssembly в 2026 году — это зрелая, production-ready технология. wasm-bindgen предоставляет бесшовный интероп между Rust и JavaScript, web-sys и js-sys дают доступ ко всем Web API, а механизмы обработки паник и аварийных завершений делают приложения надёжными. Отсутствие сборщика мусора даёт предсказуемую производительность и малый размер бинарных файлов, что делает Rust одним из лучших языков для написания WebAssembly-модулей.

В следующей главе мы рассмотрим компиляцию C и C++ в WebAssembly через Emscripten, альтернативный подход с другой моделью управления памятью и системными вызовами.
