# Глава 23. JavaScript Interop

Граница между WebAssembly и JavaScript — это критический путь в современных веб-приложениях. Каждый переход через эту границу создаёт накладные расходы, и понимание того, как работает интероп на уровне инструментов, типов, замыканий и ошибок, необходимо для эффективного использования WASM в production-средах.

В экосистеме Rust/WebAssembly основу для интеропа составляет **wasm-bindgen** — проект, который генерирует привязки между Rust-кодом и JavaScript-окружением. `wasm-bindgen` не просто обёртывает функции, а генерирует тонкий слой кода, который обрабатывает маршаллинг типов, управление памятью и интеграцию с ECMAScript-модулями . Это достигается через встраивание метаданных в кастомные секции WASM-модуля на этапе компиляции и их последующую обработку специальным линкером . Вокруг `wasm-bindgen` выстроена трёхуровневая архитектура: `js-bindgen` предоставляет базовые макросы для встраивания JS-кода, `js-sys` содержит привязки к ECMAScript API, а `web-sys` — к Web API браузера .

## Использование js-sys и web-sys в Rust

### js-sys: фундаментальные типы

`js-sys` является фундаментальной библиотекой, предоставляющей Rust-представления для всех встроенных объектов JavaScript. Центральный тип здесь — `JsValue`, который служит непрозрачным дескриптором для любого JavaScript-значения . `JsValue` управляет временем жизни через `Clone` и `Drop`, взаимодействуя с таблицей `externref` .

Кроме базового `JsValue`, `js-sys` предоставляет типизированные обёртки: `JsString`, `JsArray`, `JsNumber`, `JsBigInt`, а также `Object`, `Function`, `Promise`, `Date`, `Math`, `JSON`, `Reflect` и другие . Эти типы позволяют работать с JavaScript-значениями так же, как в самом JavaScript, но с проверкой типов на этапе компиляции Rust.

```rust
use js_sys::{Object, Reflect, Array, JSON};
use wasm_bindgen::prelude::*;

// Работа с объектом
let obj = Object::new();
Reflect::set(&obj, &"key".into(), &"value".into()).unwrap();

// Работа с массивом
let arr = Array::new();
arr.push(&1.into());
arr.push(&2.into());
arr.push(&3.into());

// Работа с JSON
let json = JSON::stringify(&obj).unwrap();
```

Важное ограничение: `js_sys::Array<T>` — это дженерик-тип, где параметр `T` существует только на уровне Rust-типов и стирается при генерации JavaScript-биндингов. Попытка некорректного upcasting, например преобразования `Array<T>` в `ArrayTuple<(...)>`, была удалена в последних версиях `wasm-bindgen`, поскольку массив не может статически доказать фиксированную арность .

### web-sys: полный доступ к Web API

`web-sys` построен поверх `js-sys` и генерируется непосредственно из WebIDL-спецификаций браузерных API . Он содержит привязки ко всем Web API: `Window`, `Document`, `console`, `Storage`, `fetch`, `WebSocket`, `CanvasRenderingContext2d`, `WebGPU` и многим другим.

Каждое `web-sys`-имя доступно как модуль, организованный по JavaScript-неймспейсам :

```rust
use web_sys::{console, window, Document, Storage};

// Вызов console.log
console::log_1(&"Hello from Rust".into());

// Доступ к окну и документу
let win = window().expect("no global window exists");
let doc = win.document().expect("should have a document");

// Работа с localStorage
let storage = win.local_storage().unwrap().unwrap();
storage.set_item("key", "value").unwrap();
```

`web-sys` отличается модульностью: включаются только те API, которые реально используются в коде, что минимизирует размер генерируемых биндингов . В 2026 году актуальные версии: `js-sys` 0.3.95, `web-sys` 0.3.95, `wasm-bindgen` 0.2.123 .

### Маршаллинг типов

`wasm-bindgen` автоматически преобразует типы на границе Rust ↔ JavaScript :

| Rust-тип | JavaScript-тип | Направление |
|----------|----------------|-------------|
| `i32` / `u32` | `number` | Оба |
| `f64` | `number` | Оба |
| `bool` | `boolean` | Оба |
| `String` | `string` | Оба |
| `&str` | `string` | Rust ← JS |
| `Vec<T>` | `Array` / `TypedArray` | Rust ← JS |
| `JsValue` | `any` | Оба |
| `*const T` | `number` (указатель) | Rust → JS |

`JsOption<T>` в `wasm-bindgen` 0.2.123 теперь трактует только `undefined` как пустое значение, что соответствует строгой TypeScript-семантике `T | undefined`. `null` теперь является **присутствующим** значением, а не отсутствующим .

## Прокси и хелперы

### Структура js-bindgen

Архитектура `js-bindgen` включает несколько слоёв, каждый из которых решает свою задачу :

- **js-bindgen** — минимальный слой, предоставляющий макросы `global_asm!` и механизмы для импорта и встраивания JavaScript. Поставляется с кастомным линкером `js-bindgen-ld`.
- **js-sys** — привязки к ECMAScript API. Макрос `#[js_sys]` упрощает генерацию биндингов как для внутреннего использования, так и для пользователей.
- **web-sys** — привязки к Web API, полностью генерируемые из WebIDL-файлов. Для сокращения времени компиляции код генерируется путём пре-экспансии макроса `#[js_sys]`.

Это разделение позволяет использовать `js-bindgen` для создания различных тулчейнов, а не только для стандартной связки Rust↔JS. Разработчики, чувствительные к размеру зависимостей, могут импортировать только `js-bindgen` без `js-sys` и `web-sys`, если им нужен только доступ к `console.log()` .

### Асинхронность и spawn_local

Для асинхронных операций в WASM-среде используется `spawn_local` из `wasm-bindgen-futures`. Он позволяет запускать Rust-футуры, интегрируя их с JavaScript-циклом событий :

```rust
use wasm_bindgen_futures::spawn_local;
use web_sys::{window, Response};

spawn_local(async move {
    let win = window().expect("no window");
    let promise = win.fetch_with_str("https://api.example.com/data");
    let future = JsFuture::from(promise);
    match future.await {
        Ok(response) => {
            let resp: Response = response.dyn_into().unwrap();
            // обработка ответа
        }
        Err(err) => {
            console::log_1(&err);
        }
    }
});
```

`spawn_local` является стандартным паттерном в Rust/WASM-фреймворках и часто ре-экспортируется для упрощения доступа .

## Передача колбэков

Передача замыканий (колбэков) из Rust в JavaScript — одна из самых сложных задач интеропа. `wasm-bindgen` предоставляет для этого тип `ScopedClosure` (с алиасом `Closure` для обратной совместимости) .

### Два режима работы

`ScopedClosure` работает в двух режимах в зависимости от времени жизни :

1. **Заимствованные замыкания** (`&ScopedClosure<'a, T>`). Используются для синхронных обратных вызовов с известным временем жизни. Подходят, когда JavaScript вызывает замыкание немедленно или в течение гарантированного периода. Замыкание остаётся под управлением Rust и освобождается при удалении `ScopedClosure` .

2. **Статические замыкания** (`ScopedClosure<'static, T>`). Используются, когда JavaScript хранит замыкание на неопределённый срок — для обработчиков событий, таймеров. Создаются через `Closure::new()` (алиас `Closure::own`). В этом режиме право собственности передаётся JavaScript, а интеграция с GC-финализаторами позволяет освободить память на стороне JS .

### Конструкторы и unwind safety

`ScopedClosure` предоставляет несколько конструкторов для разных сценариев :

| Конструктор | Поведение при панике | Требует `UnwindSafe` |
|-------------|---------------------|---------------------|
| `Closure::new` / `own` | Перехватывает, создаёт `PanicError` | Да |
| `Closure::new_aborting` | Немедленный abort | Нет |
| `Closure::new_assert_unwind_safe` | Перехватывает | Нет (проверка пропущена) |
| `Closure::once` | Перехватывает | Да |
| `Closure::once_aborting` | Немедленный abort | Нет |
| `Closure::borrow` / `borrow_mut` | Перехватывает | Да |
| `Closure::borrow_aborting` | Немедленный abort | Нет |

`_aborting`-варианты не перехватывают паники и приводят к аварийному завершению WASM-инстанса. Они используются, когда раскрутка стека невозможна из-за ограничений unwind safety .

### Модель владения

`ScopedClosure` следует той же модели владения, что и другие типы `wasm-bindgen`: JavaScript-ссылка остаётся действительной до тех пор, пока Rust-значение не будет удалено. При удалении `ScopedClosure` замыкание аннулируется, и любые последующие вызовы из JavaScript выбрасывают исключение: `"closure invoked recursively or after being dropped"` .

Для заимствованных замыканий (через `borrow`/`borrow_mut`) Rust-заимствователь гарантирует, что `ScopedClosure` не может пережить захваченные данные .

## Обработка ошибок на границе

Обработка паник и ошибок на границе Rust ↔ JavaScript в 2026 году стала значительно более зрелой благодаря поддержке `panic=unwind` в WebAssembly.

### panic=unwind для экспортированных функций

При сборке с `-Cpanic=unwind` и стандартной библиотекой (`-Zbuild-std`) паники в экспортированных Rust-функциях **перехватываются** на границе и преобразуются в JavaScript-исключения `PanicError` :

```rust
#[wasm_bindgen]
pub fn divide(a: i32, b: i32) -> i32 {
    if b == 0 {
        panic!("division by zero");
    }
    a / b
}
```

В JavaScript:
```javascript
import { divide } from './my_module.js';
try {
    divide(10, 0);
} catch (e) {
    console.log(e.name);    // "PanicError"
    console.log(e.message); // "division by zero"
}
```

Для асинхронных функций `Promise` отклоняется с `PanicError` .

### Требования для panic=unwind

Для использования перехвата паник необходимы :
- **Rust nightly** — `-Zbuild-std` доступен только на ночных сборках
- **`panic=unwind`** — стратегия паники должна быть настроена на раскрутку, а не на abort
- **`-Zbuild-std=std,panic_unwind`** — пересборка стандартной библиотеки с поддержкой unwind
- **Поддержка WebAssembly Exception Handling** — Node.js 22.22.3+ / 24.15.0+ или современный браузер

```bash
RUSTFLAGS="-Cpanic=unwind" cargo +nightly build --target wasm32-unknown-unknown -Zbuild-std=std,panic_unwind
```

### Обработка паник в замыканиях

Все замыкания (`Closure`, `ScopedClosure`) по умолчанию перехватывают паники при сборке с `panic=unwind` и преобразуют их в `PanicError` .

Для прямых `&dyn Fn` и `&mut dyn FnMut`-аргументов макрос `#[wasm_bindgen]` автоматически добавляет границу `MaybeUnwindSafe`, требуя от вызывающего кода оборачивать значения, небезопасные для раскрутки (`Cell<T>`, `&mut T`, `RefCell<T>`), в `std::panic::AssertUnwindSafe` .

### Неперехватываемые ошибки

Некоторые ошибки не могут быть перехвачены `catch_unwind`: `unreachable`-инструкции, переполнение стека, нехватка памяти. В таких случаях WASM-инстанс становится «отравленным» (poisoned), и все последующие вызовы экспортированных функций выбрасывают `"Module terminated"` .

`wasm-bindgen` предоставляет хуки для обработки таких ситуаций: `set_on_abort` для перехвата аварийных завершений, `schedule_reinit()` и `set_on_reinit` для восстановления инстанса .

### Паника в многопоточных сценариях

В многопоточных WASM-приложениях (с использованием SharedArrayBuffer) главный источник паник на границе — само по себе то, что `JsValue` не является `Send`/`Sync`: JS-объекты живут в куче одного потока (обычно главного), и попытка обратиться к ним из другого воркера — это не гонка данных в классическом смысле, а прямое нарушение однопоточной природы JS-объектов, которое либо ловится компилятором Rust на этапе типов, либо приводит к панике во время выполнения. На практике это означает: держите `JsValue`/DOM-ссылки в том потоке, где они были созданы, и передавайте между воркерами только примитивы или структуры, сериализуемые через `postMessage`/`SharedArrayBuffer`.


### Обработка ошибок с JsValue

Для контролируемой передачи ошибок на границу используется тип `Result<T, JsValue>`:

```rust
#[wasm_bindgen]
pub fn validate(data: &str) -> Result<bool, JsValue> {
    if data.is_empty() {
        return Err(JsValue::from_str("Data cannot be empty"));
    }
    // валидация
    Ok(true)
}
```

В JavaScript ошибка выбрасывается как исключение, и её можно перехватить стандартным `try/catch`.

---

JavaScript Interop в WebAssembly в 2026 году стал значительно более зрелым. `js-sys` и `web-sys` предоставляют полный доступ ко всем браузерным API, `ScopedClosure` даёт гибкое управление замыканиями с разными временами жизни и стратегиями обработки паник, а поддержка `panic=unwind` позволяет предсказуемо обрабатывать ошибки на границе двух миров. Понимание этих механизмов — необходимое условие для создания надёжных и производительных WASM-приложений.
