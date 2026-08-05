# Приложение D. Код-примеры основных паттернов

В этом приложении собраны практические примеры кода, иллюстрирующие основные паттерны работы с WebAssembly на разных языках.

## Rust + WASM

### Базовый модуль с wasm-bindgen

```rust
// Cargo.toml
[package]
name = "wasm-example"
version = "0.1.0"
edition = "2024"

[lib]
crate-type = ["cdylib"]

[dependencies]
wasm-bindgen = "0.2.118"
serde = { version = "1.0", features = ["derive"] }
serde-wasm-bindgen = "0.6"
```

```rust
// src/lib.rs
use wasm_bindgen::prelude::*;
use serde::{Deserialize, Serialize};

// Экспорт функции в JavaScript
#[wasm_bindgen]
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

// Передача строк
#[wasm_bindgen]
pub fn greet(name: &str) -> String {
    format!("Hello, {}!", name)
}

// Передача сложных структур через JSON
#[derive(Serialize, Deserialize)]
pub struct Point {
    pub x: f64,
    pub y: f64,
}

#[derive(Serialize, Deserialize)]
pub struct Line {
    pub start: Point,
    pub end: Point,
}

#[wasm_bindgen]
pub fn length(line: JsValue) -> JsValue {
    let line: Line = serde_wasm_bindgen::from_value(line).unwrap();
    let dx = line.end.x - line.start.x;
    let dy = line.end.y - line.start.y;
    let len = (dx * dx + dy * dy).sqrt();
    serde_wasm_bindgen::to_value(&len).unwrap()
}

// Работа с памятью: выделение и освобождение
#[wasm_bindgen]
pub fn alloc(size: usize) -> *mut u8 {
    let mut vec = Vec::with_capacity(size);
    let ptr = vec.as_mut_ptr();
    std::mem::forget(vec);
    ptr
}

#[wasm_bindgen]
pub unsafe fn dealloc(ptr: *mut u8, size: usize) {
    if !ptr.is_null() && size > 0 {
        drop(Vec::from_raw_parts(ptr, 0, size));
    }
}

// Обработка паник через глобальный перехватчик
#[wasm_bindgen(start)]
pub fn main() {
    std::panic::set_hook(Box::new(|info| {
        let message = if let Some(s) = info.payload().downcast_ref::<&str>() {
            s.to_string()
        } else {
            "Unknown panic".to_string()
        };
        web_sys::console::error_1(&format!("Panic: {}", message).into());
    }));
}
```

### Использование в JavaScript

```javascript
// index.js
import init, { add, greet, length, alloc, dealloc } from './pkg/wasm_example.js';

async function main() {
    await init();

    // Простые вызовы
    console.log(add(2, 3)); // 5
    console.log(greet("World")); // "Hello, World!"

    // Сложные структуры
    const line = {
        start: { x: 0.0, y: 0.0 },
        end: { x: 3.0, y: 4.0 }
    };
    console.log(length(line)); // 5.0

    // Работа с памятью: zero-copy
    const data = new Uint8Array([1, 2, 3, 4, 5]);
    const ptr = alloc(data.length);
    const view = new Uint8Array(instance.exports.memory.buffer, ptr, data.length);
    view.set(data);
    // ... вызов функций, работающих с ptr ...
    dealloc(ptr, data.length);
}

main();
```

## C++ + Emscripten

### Базовый модуль с Emscripten

```cpp
// example.cpp
#include <emscripten.h>
#include <emscripten/bind.h>
#include <string>
#include <vector>

using namespace emscripten;

// Простая функция
extern "C" int add(int a, int b) {
    return a + b;
}

// Работа со строками
extern "C" char* greet(const char* name) {
    std::string result = "Hello, " + std::string(name) + "!";
    char* ptr = (char*)malloc(result.length() + 1);
    strcpy(ptr, result.c_str());
    return ptr;
}

// Сложная структура
struct Point {
    double x;
    double y;
};

struct Line {
    Point start;
    Point end;
};

extern "C" double length(const Line* line) {
    double dx = line->end.x - line->start.x;
    double dy = line->end.y - line->start.y;
    return sqrt(dx * dx + dy * dy);
}

// Экспорт для Emscripten bindings
EMSCRIPTEN_BINDINGS(my_module) {
    function("add", &add);
    function("greet", &greet, allow_raw_pointers());
    value_object<Point>("Point")
        .field("x", &Point::x)
        .field("y", &Point::y);
    value_object<Line>("Line")
        .field("start", &Line::start)
        .field("end", &Line::end);
    function("length", &length, allow_raw_pointers());
}
```

### Сборка

```bash
em++ example.cpp -o example.js \
    -lembind \
    -sEXPORTED_FUNCTIONS='["_add","_greet","_length"]' \
    -sEXPORTED_RUNTIME_METHODS='["ccall","cwrap","getValue","setValue"]' \
    -O3
```

### Использование в JavaScript

```html
<!-- index.html -->
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>Emscripten Example</title>
</head>
<body>
    <script>
        var Module = {
            onRuntimeInitialized: function() {
                // Прямой вызов
                console.log(Module._add(2, 3)); // 5

                // Через ccall
                var result = Module.ccall(
                    'add',           // имя функции
                    'number',        // тип возврата
                    ['number', 'number'], // типы аргументов
                    [2, 3]           // значения
                );
                console.log(result); // 5

                // Работа со строками через cwrap
                var greet = Module.cwrap('greet', 'string', ['string']);
                console.log(greet('World')); // "Hello, World!"

                // Работа со структурами
                var linePtr = Module._malloc(32); // sizeof(Line) ~ 32 байт
                Module.setValue(linePtr + 0, 0.0, 'double');   // start.x
                Module.setValue(linePtr + 8, 0.0, 'double');   // start.y
                Module.setValue(linePtr + 16, 3.0, 'double');  // end.x
                Module.setValue(linePtr + 24, 4.0, 'double');  // end.y

                var len = Module.ccall('length', 'number', ['number'], [linePtr]);
                console.log(len); // 5.0

                Module._free(linePtr);
            }
        };
    </script>
    <script src="example.js"></script>
</body>
</html>
```

## AssemblyScript

### Базовый модуль

```typescript
// src/index.ts
// Низкоуровневый подход — работа с сырой памятью
import { memory } from "wasm";

// Экспорт функции для аллокации памяти
export function alloc(size: i32): i32 {
    return memory.data(size);
}

// Экспорт функции для освобождения памяти
export function free(ptr: i32, size: i32): void {
    memory.free(ptr, size);
}

// Простая функция
export function add(a: i32, b: i32): i32 {
    return a + b;
}

// Работа со строками
export function greet(name: string): string {
    return "Hello, " + name + "!";
}

// Работа с массивами через высокоуровневый API
export function sumArray(arr: Int32Array): i32 {
    let sum = 0;
    for (let i = 0; i < arr.length; i++) {
        sum += arr[i];
    }
    return sum;
}

// Низкоуровневая работа с памятью
export function processBuffer(ptr: i32, len: i32): i32 {
    let sum = 0;
    for (let i = 0; i < len; i++) {
        sum += load<i32>(ptr + i * 4);
    }
    return sum;
}

// Комплексная структура
class Point {
    constructor(public x: f64, public y: f64) {}
}

export function distance(a: Point, b: Point): f64 {
    let dx = b.x - a.x;
    let dy = b.y - a.y;
    return Math.sqrt(dx * dx + dy * dy);
}
```

### Сборка и использование

```bash
# Установка AssemblyScript
npm init -y
npm install --save-dev assemblyscript

# Сборка
npx asc src/index.ts --outFile build/index.wasm --optimize --bindings esm
```

```javascript
// index.js
import { add, greet, sumArray, distance, Point } from './build/index.js';

console.log(add(2, 3)); // 5
console.log(greet('World')); // "Hello, World!"

const arr = new Int32Array([1, 2, 3, 4, 5]);
console.log(sumArray(arr)); // 15

const p1 = new Point(0, 0);
const p2 = new Point(3, 4);
console.log(distance(p1, p2)); // 5.0
```

## JavaScript Interop

### WASM-модуль с импортами и экспортами (WAT)

```wasm
;; module.wat
(module
  ;; Импорт функции log из окружения
  (import "env" "log" (func $log (param i32)))
  (import "env" "memory" (memory 1))

  ;; Экспорт функции add
  (func $add (export "add") (param i32 i32) (result i32)
    local.get 0
    local.get 1
    i32.add
  )

  ;; Экспорт функции, вызывающей импортированную log
  (func $logAndAdd (export "logAndAdd") (param i32 i32) (result i32)
    local.get 0
    call $log
    local.get 1
    call $log
    local.get 0
    local.get 1
    i32.add
  )

  ;; Функция, работающая с памятью
  (func $writeMemory (export "writeMemory") (param $ptr i32) (param $value i32)
    local.get $ptr
    local.get $value
    i32.store
  )

  (func $readMemory (export "readMemory") (param $ptr i32) (result i32)
    local.get $ptr
    i32.load
  )
)
```

### JavaScript-хост

```javascript
// host.js
// Сборка WAT в WASM
const wasmBytes = await fetch('module.wasm');
const { instance } = await WebAssembly.instantiateStreaming(wasmBytes, {
    env: {
        log: (value) => console.log('WASM log:', value),
        memory: new WebAssembly.Memory({ initial: 1 })
    }
});

// Вызов экспортированных функций
console.log(instance.exports.add(2, 3)); // 5
instance.exports.logAndAdd(2, 3); // выводит "WASM log: 2", "WASM log: 3", возвращает 5

// Работа с памятью: zero-copy
const memory = instance.exports.memory;
const ptr = 0;
instance.exports.writeMemory(ptr, 42);
const value = instance.exports.readMemory(ptr);
console.log(value); // 42

// Использование типизированных массивов для доступа к памяти
const view = new Int32Array(memory.buffer);
view[0] = 100;
console.log(instance.exports.readMemory(0)); // 100
```

### Передача колбэков через wasm-bindgen (Rust)

```rust
// src/lib.rs
use wasm_bindgen::prelude::*;

// Тип для колбэка (экспортируется в JS)
#[wasm_bindgen]
extern "C" {
    pub type Callback;
    #[wasm_bindgen(method)]
    pub fn call(this: &Callback, value: i32);
}

// Функция, принимающая колбэк
#[wasm_bindgen]
pub fn process_with_callback(callback: &Callback) {
    for i in 0..10 {
        callback.call(i);
    }
}

// Замыкание, переданное в JS
#[wasm_bindgen]
pub fn create_closure() -> js_sys::Function {
    let closure = Closure::wrap(Box::new(|value: i32| {
        web_sys::console::log_1(&format!("Value: {}", value).into());
    }) as Box<dyn Fn(i32)>);
    closure.into_js_value().unchecked_into()
}
```

```javascript
// JavaScript использование
import init, { process_with_callback, create_closure } from './pkg/wasm_example.js';

await init();

// Передача колбэка как объекта с методом call
const callback = {
    call: (value) => console.log('Callback received:', value)
};
process_with_callback(callback);

// Передача замыкания
const closure = create_closure();
closure(42); // "Value: 42"
closure(100); // "Value: 100"
```

### Передача больших данных без копирования (Zero-copy)

```rust
// Rust-сторона
#[wasm_bindgen]
pub fn process_data(ptr: *const u8, len: usize) -> i32 {
    let data = unsafe { std::slice::from_raw_parts(ptr, len) };
    data.iter().map(|&x| x as i32).sum()
}
```

```javascript
// JavaScript-сторона
// Выделяем буфер в WASM-памяти
const ptr = instance.exports.alloc(1024);
const view = new Uint8Array(memory.buffer, ptr, 1024);
view.set(new Uint8Array([1, 2, 3, 4, 5]));

// Вызываем функцию — данные не копируются
const sum = instance.exports.process_data(ptr, 5);
console.log(sum); // 15

// Освобождаем память
instance.exports.dealloc(ptr, 1024);
```

### Использование Component Model и WIT (Rust)

```wit
// wit/example.wit
package example:calculator;

interface types {
    record point {
        x: f64,
        y: f64,
    }

    record line {
        start: point,
        end: point,
    }
}

world calculator {
    import types;

    // Импорт из окружения
    import log: func(value: string);

    // Экспорт для окружения
    export add: func(a: s32, b: s32) -> s32;
    export greet: func(name: string) -> string;
    export length: func(line: types/line) -> f64;
}
```

```rust
// src/lib.rs
wit_bindgen::generate!("calculator");

use calculator::types::Line;

struct Calculator;

impl Guest for Calculator {
    fn add(a: i32, b: i32) -> i32 {
        a + b
    }

    fn greet(name: String) -> String {
        format!("Hello, {}!", name)
    }

    fn length(line: Line) -> f64 {
        let dx = line.end.x - line.start.x;
        let dy = line.end.y - line.start.y;
        (dx * dx + dy * dy).sqrt()
    }
}

export!(Calculator);
```