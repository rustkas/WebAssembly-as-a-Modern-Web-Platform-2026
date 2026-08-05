# Глава 18. JavaScript ↔ WebAssembly

WebAssembly не существует в изоляции. Его главная сила — возможность взаимодействовать с JavaScript и веб-платформой. Это взаимодействие происходит в обе стороны: JavaScript вызывает функции WebAssembly, а WebAssembly вызывает функции JavaScript . Понимание этого интеропа — ключевое условие эффективного использования WASM в реальных приложениях.

## Вызов JS-функций из WASM

WebAssembly-модули могут импортировать функции из окружения — в браузере это функции JavaScript . Импорты объявляются в секции импортов модуля с двухуровневым пространством имён: имя модуля и имя элемента внутри этого модуля.

Пример модуля, импортирующего JS-функцию `console.log`:

```wat
(module
  (func $log (import "js" "console.log") (param i32))
  (func (export "callLog")
    i32.const 42
    call $log
  )
)
```

При инстанцировании модуля окружение должно предоставить объект с реализациями всех импортов :

```javascript
const importObject = {
  js: {
    "console.log": (arg) => console.log(arg)
  }
};

const instance = await WebAssembly.instantiate(bytes, importObject);
instance.exports.callLog(); // выведет 42
```

### Встроенные функции JavaScript (JavaScript Builtins)

В 2026 году появилась поддержка встроенных функций JavaScript — WASM-эквивалентов операций JavaScript, которые предоставляются без необходимости импортировать JS-глюк-код . Встроенные функции решают проблему производительности: импорт функций через обычный `import` создаёт накладные расходы на косвенные вызовы и преобразования `this`, в то время как встроенные функции компилируются в оптимальный код .

Пример использования строковых встроенных функций:

```wat
(module
  (func $concat (import "wasm:js-string" "concat")
    (param externref externref) (result (ref extern)))
  (func $log (import "js" "console.log") (param externref))
  (func (export "main")
    (call $log (call $concat (global.get $hello) (global.get $world))))
)
```

Встроенные функции включаются при компиляции через опцию `builtins`:

```javascript
WebAssembly.instantiate(bytes, importObject, {
  builtins: ["js-string"]
});
```

Доступные встроенные функции включают операции со строками: `concat`, `compare`, `equals`, `charCodeAt`, `codePointAt`, `length`, `substring`, а также функции преобразования `cast`, `fromCharCode`, `intoCharCodeArray` .

## Вызов WASM-функций из JS

JavaScript вызывает WebAssembly-функции тремя способами: через прямой доступ к экспортам, через Emscripten-хелперы `ccall`/`cwrap` или через низкоуровневый доступ к линейной памяти.

### Прямой вызов через экспорты

Самый простой способ. После инстанцирования модуля экспортированные функции доступны через свойство `exports` инстанса :

```javascript
const { instance } = await WebAssembly.instantiate(bytes, importObject);
const result = instance.exports.add(2, 3); // 5
```

Функция вызывается как обычная JavaScript-функция. Аргументы автоматически преобразуются в WASM-типы (`number` → `i32`/`f64`, `string` не поддерживается напрямую — требует работы с памятью).

### Emscripten: ccall и cwrap

Emscripten предоставляет хелперы `ccall` и `cwrap`, которые упрощают вызов функций, особенно с передачей строк и массивов :

**`ccall`** — выполняет функцию немедленно:

```javascript
const result = Module.ccall(
  'Add',                  // имя функции (без ведущего подчёркивания)
  'number',               // тип возвращаемого значения
  ['number', 'number'],   // типы параметров
  [1, 2]                  // значения параметров
);
```

Типы параметров могут быть `'number'`, `'string'`, `'array'`. Для строк и массивов `ccall` автоматически выделяет память, копирует данные и освобождает память после вызова .

**`cwrap`** — возвращает JavaScript-функцию, которую можно вызывать многократно:

```javascript
const add = Module.cwrap('Add', 'number', ['number', 'number']);
const result = add(4, 5); // 9
```

Важное ограничение: при использовании `'string'` или `'array'` Emscripten выделяет временную память, которая освобождается сразу после возврата. Если WASM-модуль сохраняет указатель для будущего использования, данные могут стать невалидными .

### Прямой вызов без хелперов

Для случаев, где важна производительность, функция может быть вызвана напрямую через свойство `Module._FunctionName` :

```javascript
const result = Module._Add(2, 5); // 7
```

Этот подход быстрее, но требует ручного управления памятью для строк и массивов: выделение через `_malloc`, запись в `HEAP*`, вызов функции, освобождение через `_free`.

## Передача сложных объектов без копирования (Zero-copy)

Граница между JavaScript и WebAssembly — это барьер, пересечение которого стоит дорого . Каждый переход, особенно с передачей данных, создаёт накладные расходы. Zero-copy — это техника, позволяющая обмениваться данными без лишнего копирования.

### Механизм zero-copy

WebAssembly использует линейную память — непрерывный массив байтов, доступный и WASM-коду, и JavaScript через `ArrayBuffer` . JavaScript может читать и писать в этот буфер напрямую, без копирования данных.

В Rust-экосистеме zero-copy реализуется через структуры, управляющие буферами в линейной памяти :

```rust
pub struct WasmBuffer {
    data: Vec<u8>,
    capacity: usize,
}

impl WasmBuffer {
    pub fn as_ptr(&self) -> *const u8 {
        self.data.as_ptr()
    }
    pub fn len(&self) -> usize {
        self.data.len()
    }
}
```

JavaScript получает указатель и длину, создаёт `Uint8Array` поверх WASM-памяти без копирования:

```javascript
const ptr = instance.exports.get_buffer_ptr();
const len = instance.exports.get_buffer_len();
const view = new Uint8Array(memory.buffer, ptr, len);
// view — это представление данных в WASM-памяти, без копирования
```

### Цена пересечения границы

Исследования показывают относительную стоимость операций на границе Rust↔JS :

| Операция | Относительная стоимость |
|----------|------------------------|
| Rust → Rust | 1× |
| JS → Rust (примитивы) | ~10× |
| JS → Rust (копирование строк/массивов) | ~50× |
| JS → Rust (сериализация через serde) | ~100× |
| Rust → DOM (через web-sys) | ~200× |

**Ключевое правило**: группировать работу на стороне WASM и минимизировать количество переходов. Не пересекать границу в плотных циклах .

### Буферные пулы

Для повторяющихся операций (анимация, потоковая обработка) используются буферные пулы — переиспользуемые буферы, избегающие аллокации на каждом кадре :

```rust
pub struct BufferPool {
    buffers: Vec<WasmBuffer>,
    max_buffers: usize,
}

impl BufferPool {
    pub fn acquire(&mut self, capacity: usize) -> WasmBuffer;
    pub fn release(&mut self, buf: WasmBuffer);
}
```

Буферный пул поддерживает до `max_buffers` буферов, переиспользуя их вместо повторного выделения памяти .

### Jco и транспиляция компонентов

Jco (Bytecode Alliance) — это "мульти-инструмент для JS-экосистемы WebAssembly" . Он позволяет взять любой WASM-компонент (написанный на Rust, Go, Python, C, JavaScript или другом языке) и преобразовать его в core Wasm модули с JS-глюк-кодом, работающим в Node.js или браузере без нативной поддержки Component Model .

**jco transpile** выполняет:
1. Разбор компонента на составляющие core Wasm модули.
2. Генерацию кода подъёма/опускания для каждого типа Component Model на основе WIT-контракта через `js-component-bindgen` .
3. Добавление WASI P2/P3-шеймов — JavaScript-реализаций WASI-интерфейсов (файловая система, HTTP, часы, случайность) .

## Типичные паттерны интеропа

### Передача строк

Строки передаются либо через Emscripten-хелперы (удобно, но с временным выделением памяти), либо через ручное управление буферами в линейной памяти.

**Ручной подход**:
```rust
#[wasm_bindgen]
pub fn process_string(data_ptr: *const u8, data_len: usize) -> *const u8 {
    let slice = unsafe { std::slice::from_raw_parts(data_ptr, data_len) };
    let string = std::str::from_utf8(slice).unwrap();
    let result = format!("Processed: {}", string);
    // Возврат указателя на результат
    let result_ptr = result.as_ptr();
    std::mem::forget(result); // не освобождать
    result_ptr
}
```

### Коллбэки

Передача функций обратного вызова из JavaScript в WASM:

```rust
#[wasm_bindgen]
extern "C" {
    fn on_complete(result: i32);
}

#[wasm_bindgen]
pub fn do_work() {
    // выполнить работу
    on_complete(42);
}
```

### Передача объектов через JsValue

Для передачи сложных объектов используется `JsValue` — opaque-тип, представляющий любое JavaScript-значение :

```rust
#[wasm_bindgen]
pub fn process(data: JsValue) -> JsValue {
    // data — объект из JavaScript
    // Для работы с ним требуется преобразование
    let obj = data.dyn_into::<js_sys::Object>().unwrap();
    let value = js_sys::Reflect::get(&obj, &"key".into()).unwrap();
    value
}
```

Для структурированных данных используется `serde_wasm_bindgen`:

```rust
#[derive(Serialize, Deserialize)]
struct Metrics { sum: i64, count: usize }

#[wasm_bindgen]
pub fn analyze(data: &[i32]) -> JsValue {
    let result = Metrics { sum: data.iter().sum(), count: data.len() };
    serde_wasm_bindgen::to_value(&result).unwrap()
}
```

### Дженерики в wasm-bindgen

wasm-bindgen поддерживает дженерики для типов JavaScript с помощью type erasure: дженерик-параметры существуют только в Rust-коде и стираются при генерации JS-биндингов .

```rust
#[wasm_bindgen]
pub fn process_array(arr: &js_sys::Array<js_sys::Number>) -> f64 {
    arr.iter().map(|n| n.as_f64().unwrap()).sum()
}
```

Трейт `JsGeneric` используется для ограничений: типы, реализующие `JsGeneric`, стираются в `JsValue` при генерации биндингов . **Rust-примитивы** (`u32`, `f64`, `String`) не реализуют `JsGeneric` — их нельзя использовать как параметры дженериков для JS-типов. Вместо `Array<u32>` нужно использовать `Array<Number>` .

**Upcasting** — безопасное преобразование между типами :

```rust
let num_array: Array<Number> = Array::new_typed();
let js_array: Array<JsValue> = num_array.upcast_into(); // Array<Number> → Array<JsValue>
let obj: Object = num_array.upcast_into(); // Array<Number> → Object (наследование)
```

Upcast-реализации генерируются автоматически для всех импортированных JS-типов на основе атрибутов `extends` .

### Variadic-функции

Для функций JavaScript с переменным числом аргументов (variadic) используется атрибут `#[wasm_bindgen(variadic)]` :

```rust
#[wasm_bindgen]
extern "C" {
    #[wasm_bindgen(js_namespace = console, variadic)]
    fn log(args: &[JsValue]);
}

// Использование
log(&[JsValue::from("Hello"), JsValue::from(42), JsValue::TRUE]);
```

Экспорт Rust-функции с variadic в JavaScript:
```rust
#[wasm_bindgen(variadic)]
pub fn variadic_function(arr: &JsValue) -> JsValue {
    arr.into()
}
// В TypeScript: export function variadic_function(...arr: any): any;
```

---

Эффективный интероп между JavaScript и WebAssembly требует понимания нескольких ключевых принципов: минимизация переходов через границу, использование zero-copy для больших данных, группировка операций на стороне WASM. Инструменты вроде wasm-bindgen, Emscripten, Jco и встроенные функции JavaScript делают интероп удобным, но производительность критических путей всё ещё требует ручного контроля над памятью и буферными пулами .