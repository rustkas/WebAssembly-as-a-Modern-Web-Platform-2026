# Глава 30. WASM + Web Workers

Web Workers позволяют выполнять JavaScript-код в фоновых потоках, не блокируя основной поток, отвечающий за рендеринг и взаимодействие с пользователем . Когда WebAssembly используется внутри Web Worker, вычисления выполняются в фоновом потоке, а UI остаётся отзывчивым . Вместе эти технологии дают и параллелизм (отдельный поток), и вычислительную эффективность (WASM-код) .

## Тяжёлые фоновые задачи

Главная причина использования WASM внутри Web Workers — выполнение тяжёлых вычислений без блокировки UI. Без Web Workers долгая операция на главном потоке приводит к появлению сообщения о зависшем скрипте — браузер предлагает пользователю прервать выполнение .

### Обработка изображений: 23× ускорение

Обработка 4K-изображений (4096×2160) показывает масштаб ускорения, которое даёт связка WASM + Web Workers :

| Конфигурация | Время обработки | Блокировка главного потока | Память |
|--------------|----------------|---------------------------|--------|
| Чистый JavaScript | 1850 мс | Серьёзная | 350 MB |
| WASM (один поток) | 420 мс | Заметная | 210 MB |
| WASM + 1 Worker | 150 мс | Незначительная | 230 MB |
| WASM + 4 Workers | 38 мс | Отсутствует | 260 MB |
| WASM + SIMD + 4 Workers | 22 мс | Отсутствует | 260 MB |

WASM даёт 4× ускорение по сравнению с чистым JavaScript в однопоточном режиме. Добавление одного Web Worker ускоряет обработку ещё в 2.8×. Четыре воркера с SIMD дают суммарное ускорение в **84 раза** по сравнению с чистым JavaScript .

### Практическая реализация: обработка изображений

Веб-воркер загружает WASM-модуль, получает данные через `postMessage`, выполняет обработку и возвращает результат :

```javascript
// worker.js
let wasmModule;

async function loadWasm() {
    const { instance } = await WebAssembly.instantiateStreaming(
        fetch('processor.wasm'),
        { env: { memory: new WebAssembly.Memory({ initial: 256 }) } }
    );
    wasmModule = instance.exports;
}

self.onmessage = async ({ data }) => {
    if (data.type === 'init') {
        await loadWasm();
        return;
    }

    const { buffer, width, height, operation } = data;
    const ptr = wasmModule.get_buffer(width * height * 4);
    const heap = new Uint8Array(wasmModule.memory.buffer);

    // Нулевое копирование — запись данных напрямую в WASM-память
    heap.set(new Uint8Array(buffer), ptr);

    // Выполнение обработки в WASM (использует SIMD)
    wasmModule.process(ptr, width, height, operation);

    // Возврат результата с передачей владения буфером (Transferable)
    self.postMessage({ buffer: heap.buffer }, [heap.buffer]);
};
```

**Ключевое правило для передачи данных:** использовать `postMessage` с `Transferable` — список объектов для передачи владения без копирования . Это даёт нулевое копирование при передаче данных между воркером и основным потоком.

### .NET и Blazor: C# в фоновом потоке

.NET WebAssembly-приложения (включая Blazor WebAssembly) могут запускать C#-код в Web Workers . Это позволяет выносить тяжёлые вычисления из UI-потока без переписывания логики на JavaScript.

Процесс работы :
1. Воркер инициализирует .NET WASM-рантайм
2. Основной поток отправляет команду (например, `generateQR`)
3. Воркер выполняет C#-метод с атрибутом `[JSExport]`
4. Результат возвращается через `postMessage`

```csharp
// WorkerMethods.cs
[JSExport]
internal static byte[] GenerateQR(string text, int size)
{
    var qrGenerator = new QRCodeGenerator();
    var qrData = qrGenerator.CreateQrCode(text, QRCodeGenerator.ECCLevel.Q);
    return new BitmapByteQRCode(qrData).GetGraphic(size);
}
```

```javascript
// worker.js
self.addEventListener('message', async function(e) {
    const { command, text, size, requestId } = e.data;
    if (command === 'generateQR') {
        const result = assemblyExports.QRGenerator.Generate(text, size);
        self.postMessage({ command: 'response', requestId, result });
    }
});
```

Исследования указывают, что оптимизация производительности для .NET-приложений на Web Workers включает минимизацию передачи данных, группировку операций в пакеты, контроль памяти в WASM-среде и предпочтение персистентных воркеров из-за затрат на инициализацию WASM-рантайма .

### Async-задачи в Rust: async_wasm_task

Для Rust-разработчиков библиотека `async_wasm_task` предоставляет знакомый паттерн `tokio::task` для асинхронных задач в браузере . Она позволяет выполнять задачи на Web Workers с использованием знакомого API `spawn` и `spawn_blocking`.

```rust
use async_wasm_task::{spawn, spawn_blocking};

// Асинхронная задача в текущем потоке
let handle = spawn(async {
    // Вычисление с возможностью await
    let result = compute().await;
    result
});

// Блокирующая задача на отдельном воркере
let blocking_handle = spawn_blocking(move || {
    // Синхронная тяжёлая операция
    expensive_cpu_bound_work()
});
```

Функция `spawn_blocking` автоматически управляет пулом воркеров: при необходимости создаются новые воркеры до лимита (512 по умолчанию), а неиспользуемые завершаются через 10 секунд простоя . Это напоминает модель `tokio` и позволяет писать асинхронный код в браузере с привычной семантикой.

## Коммуникация между потоками

Коммуникация между основным потоком и Web Worker построена на асинхронной передаче сообщений через `postMessage` .

### Zero-copy через Transferable

По умолчанию `postMessage` копирует данные (структурированное клонирование). Для больших данных это дорого. Использование `Transferable` передаёт владение буфером без копирования :

```javascript
// ❌ Копирование данных — медленно для больших буферов
worker.postMessage({ data: new Uint8Array(buffer) });

// ✅ Zero-copy — передача владения буфером
worker.postMessage({ buffer }, [buffer]);
```

Когда буфер передан через Transferable, основной поток теряет к нему доступ. Это эффективно, но требует аккуратного управления — данные нельзя использовать после передачи.

### Двунаправленная коммуникация

```javascript
// Основной поток
const worker = new Worker('worker.js', { type: 'module' });

// Ожидание ответа через Promise
function callWorker(command, data) {
    return new Promise((resolve, reject) => {
        const id = ++requestId;
        pendingRequests.set(id, { resolve, reject });
        worker.postMessage({ id, command, data });
    });
}

worker.onmessage = (event) => {
    const { id, result, error } = event.data;
    const pending = pendingRequests.get(id);
    if (pending) {
        pendingRequests.delete(id);
        if (error) pending.reject(new Error(error));
        else pending.resolve(result);
    }
};
```

```javascript
// Воркер
self.onmessage = async (event) => {
    const { id, command, data } = event.data;
    try {
        const result = await handleCommand(command, data);
        self.postMessage({ id, result });
    } catch (error) {
        self.postMessage({ id, error: error.message });
    }
};
```

Этот паттерн с идентификаторами запросов даёт асинхронный обмен с ожиданием ответа, сохраняя порядок сообщений.

### R в браузере через WebR

**WebR** инициализируется запуском R, скомпилированного в WebAssembly, в Web Worker . R выполняет длительные вычисления без блокировки главного потока.

Интерфейс использует асинхронные очереди сообщений и Promise API :

```javascript
// Обработка вывода в асинхронном цикле
async function run() {
    for (;;) {
        const output = await webR.read();
        switch (output.type) {
            case 'stdout': console.log(output.data); break;
            case 'stderr': console.error(output.data); break;
            case 'prompt': // R ждёт ввод
                webR.writeConsole('2 + 2');
                break;
        }
    }
}
```

### Emscripten: Pthreads vs Wasm Workers

Emscripten предоставляет два API для многопоточности: POSIX Threads (Pthreads) и Wasm Workers .

**Pthreads** ориентирован на портируемость: он эмулирует поведение нативных Pthreads, упрощая перенос кодовых баз между Linux x64 и WebAssembly. Pthreads используют кешированный пул воркеров для поддержки синхронного старта потоков, имеют прокси-очереди для вызова JS-функций (`MAIN_THREAD_EM_ASM*`), поддерживают POSIX-отмену и точки отмены .

**Wasm Workers** даёт прямое отображение на веб-примитивы . Воркеры всегда стартуют асинхронно, не имеют встроенного проксирования JS-функций, иерархичны (воркер может создавать дочерние воркеры), используют собственные примитивы синхронизации из `emscripten/wasm_worker.h`, а не Pthread-мьютексы .

Ключевые различия:
- Pthreads поддерживают синхронный `pthread_create` через пул воркеров; Wasm Workers — только асинхронный старт .
- Pthreads могут проксировать функции на главный поток; Wasm Workers требуют ручной реализации .
- Pthreads дают максимальную совместимость с нативным кодом; Wasm Workers дают меньший размер кода и лучшую производительность для специфичных WASM-приложений .

Выбор между ними: для портирования кодовых баз — Pthreads; для новых WASM-приложений — Wasm Workers .

### OffscreenCanvas для рендеринга в воркере

Web Workers не могут обращаться к DOM, но могут управлять canvas через `OffscreenCanvas` . Это позволяет рендерить графику в Web Worker без блокировки главного потока. Пример: wgpu-приложение, где воркер владеет OffscreenCanvas, управляет циклом рендеринга через `requestAnimationFrame` и рендерит через WebGPU, а основной поток только передаёт canvas и события .

## Архитектурные шаблоны

### Пул воркеров и троттлинг

Для балансировки нагрузки между воркерами используется пул, который отслеживает занятость и распределяет задачи динамически:

```javascript
class WorkerPool {
    constructor(workerScript, size) {
        this.workers = Array.from({ length: size }, () => ({
            worker: new Worker(workerScript, { type: 'module' }),
            busy: false,
        }));
        this.queue = [];
        this.requestId = 0;
        this.pending = new Map();

        this.workers.forEach(({ worker }) => {
            worker.onmessage = (event) => this.handleMessage(event);
        });
    }

    dispatch(task, data) {
        return new Promise((resolve, reject) => {
            const id = ++this.requestId;
            this.pending.set(id, { resolve, reject });

            const available = this.workers.find(w => !w.busy);
            if (available) {
                available.busy = true;
                available.worker.postMessage({ id, task, data });
            } else {
                this.queue.push({ id, task, data });
            }
        });
    }

    handleMessage(event) {
        const { id, result, error } = event.data;
        const pending = this.pending.get(id);
        if (pending) {
            this.pending.delete(id);
            if (error) pending.reject(new Error(error));
            else pending.resolve(result);
        }

        // Пометить воркер как свободный и обработать очередь
        const worker = this.workers.find(w => w.worker === event.target);
        if (worker) {
            worker.busy = false;
            if (this.queue.length > 0) {
                const next = this.queue.shift();
                worker.busy = true;
                worker.worker.postMessage(next);
            }
        }
    }
}
```

### Разделение данных на чанки

Крупные задачи разбиваются на независимые части (чанки), которые распределяются между воркерами. Каждый воркер обрабатывает свой диапазон, а результаты собираются и объединяются:

```javascript
function scheduleTiles(width, height, tileSize) {
    const tiles = [];
    for (let y = 0; y < height; y += tileSize) {
        for (let x = 0; x < width; x += tileSize) {
            tiles.push({
                x, y,
                width: Math.min(tileSize, width - x),
                height: Math.min(tileSize, height - y),
            });
        }
    }
    return tiles;
}
```

### Предзагрузка (pre-fetch) WASM-модуля

WASM-модуль загружается и компилируется в воркере, а затем передаётся на главный поток :

1. Воркер получает и компилирует `.wasm` через `compileStreaming`
2. Воркер отправляет скомпилированный `WebAssembly.Module` на главный поток через `postMessage`
3. Главный поток создаёт инстанс модуля с импортами
4. `WebAssembly.Module` поддерживает структурное клонирование и может быть передан между воркерами и основным потоком 

Это позволяет начать компиляцию в фоне до того, как главный поток готов к инстанцированию, сокращая общее время до интерактивности.

### Прогрев воркеров

Воркеры имеют затраты на запуск. Для критических по времени задач воркеры предварительно инициализируются (прогреваются) до получения реальных данных :

```javascript
// Прогрев воркера
const warmupWorker = new Worker('processor.js', { type: 'module' });
warmupWorker.postMessage({ type: 'init' });
// Загрузка WASM происходит в фоне до получения первой задачи
```

---

WASM + Web Workers — это стандартный архитектурный паттерн для высокопроизводительных веб-приложений. Web Workers обеспечивают параллелизм без блокировки UI. WebAssembly даёт вычислительную эффективность. Вместе они позволяют использовать многоядерные процессоры .

Ключевые практики: минимизация передачи данных через Transferable и OffscreenCanvas, балансировка нагрузки через пул воркеров, прогрев воркеров для снижения стартовой задержки, и разбиение задач на чанки для параллельной обработки . Выбор между Pthreads и Wasm Workers в Emscripten определяется приоритетами портируемости vs размера кода и производительности .