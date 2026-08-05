# WebAssembly as a Modern Web Platform 2026

## Семантика, архитектура и производительность современного WebAssembly

> Книга о WebAssembly как фундаментальной технологии Web Platform.  
> Не просто бинарный формат, а второй исполнительный слой веба, изменивший подход к высокопроизводительным вычислениям в браузере.

---

## 📖 О книге

За последние годы WebAssembly сильно изменился.

Сегодня WebAssembly — это уже не просто «способ запускать C++ в браузере». Это полноценная виртуальная машина, интегрированная в Web Platform:

- высокопроизводительные вычисления на уровне нативного кода;
- многопоточность и Shared Memory;
- поддержка SIMD-инструкций;
- интеграция с WebGPU;
- стандартизированный системный интерфейс (WASI);
- Component Model для композиции модулей.

Современные фреймворки:

- React
- Angular
- Vue
- Svelte
- Next.js
- Astro

не заменяют WebAssembly.

Они дополняют его, используя WASM для задач, где JavaScript достигает своих пределов — обработка изображений, аудио, видео, криптография, симуляции, игры, машинное обучение.

Эта книга показывает WebAssembly как часть современной Web Platform и как архитектурный инструмент для построения высокопроизводительных веб-приложений.

---

## 🎯 Цель книги

После изучения книги читатель должен понимать:

- что такое WebAssembly и зачем он нужен в 2026 году;
- как WASM встраивается в архитектуру браузера;
- как компилировать разные языки в WASM;
- как загружать, инстанцировать и отлаживать WASM-модули;
- как проектировать архитектуру приложений с WASM;
- как использовать многопоточность, SIMD и WebGPU;
- как развертывать WASM в production;
- как устроен Component Model и WASI;
- куда движется WebAssembly в ближайшие годы.

---

## 👥 Для кого эта книга

Книга предназначена для:

- Frontend-разработчиков;
- Fullstack-разработчиков;
- архитекторов интерфейсов;
- разработчиков игр и визуализаций;
- инженеров по производительности (Web Performance);
- разработчиков системного ПО, переходящих в веб;
- всех, кто хочет понимать будущее веб-платформы.

---

## 🧠 Главная идея

Современный frontend строится на двух исполнительных слоях:

```
Web Platform Execution Layers

┌─────────────────────────────────────┐
│          JavaScript                 │
│  ───────────────────────────────   │
│  • Динамическая типизация          │
│  • Сборка мусора                   │
│  • Событийно-ориентированная       │
│  • DOM-манипуляции                 │
└─────────────────────────────────────┘
                   +
┌─────────────────────────────────────┐
│         WebAssembly                 │
│  ───────────────────────────────   │
│  • Статическая типизация           │
│  • Линейная память                 │
│  • Компилируемость                 │
│  • Высокая производительность      │
└─────────────────────────────────────┘
```

WebAssembly не заменяет JavaScript.  
Они работают **вместе**, каждый в своей зоне ответственности:

```
JavaScript ──→ отвечает за взаимодействие с пользователем, DOM, события

WebAssembly ──→ отвечает за вычисления, обработку данных, бизнес-логику
```

---

## 📚 Содержание

### Предисловие

## Эпоха пост-JavaScript: зачем вебу второй исполнительный слой

- От asm.js к WebAssembly: эволюция вычислений в браузере
- Почему JIT-компиляция JavaScript не решает все проблемы
- WebAssembly как системный уровень Web Platform
- Статистика внедрения WASM в 2026 году
- Что изменилось за последние пять лет

* [📖 Читать главу](./book/preface.md)  
* [📚 Литература](./references/preface.md)  
* [💻 Примеры](./examples/preface.md)  
* [🧪 Практика](./exercises/preface.md)

---

### Часть I. WebAssembly после JavaScript

## Глава 1. Что такое WebAssembly в 2026

Темы:

- Определение и спецификация WASM
- WASM как бинарный формат и виртуальная машина
- Место WASM в стеке веб-технологий
- Основные сценарии использования

* [📖 Читать главу](./book/chapter-01.md)  
* [📚 Литература](./references/chapter-01.md)  
* [💻 Примеры](./examples/chapter-01.md)  
* [🧪 Практика](./exercises/chapter-01.md)

---

## Глава 2. Почему WebAssembly появился

Темы:

- Ограничения JavaScript для вычислительных задач
- История: PNaCl, asm.js, Emscripten
- Проблема детерминизма производительности
- Потребность в переносимости бинарного кода

* [📖 Читать главу](./book/chapter-02.md)  
* [📚 Литература](./references/chapter-02.md)  
* [💻 Примеры](./examples/chapter-02.md)  
* [🧪 Практика](./exercises/chapter-02.md)

---

## Глава 3. WASM vs JavaScript

Темы:

- Компилируемость против интерпретируемости
- Статическая против динамической типизации
- Линейная память против управляемой кучи
- Производительность: что и когда быстрее
- Когда использовать WASM, а когда JS

* [📖 Читать главу](./book/chapter-03.md)  
* [📚 Литература](./references/chapter-03.md)  
* [💻 Примеры](./examples/chapter-03.md)  
* [🧪 Практика](./exercises/chapter-03.md)

---

## Глава 4. WASM vs Web Workers

Темы:

- Модели параллелизма в вебе
- Workers как изоляция потока ОС
- WASM как изоляция вычислений
- Совместное использование: WASM внутри Worker
- SharedArrayBuffer и атомарные операции

* [📖 Читать главу](./book/chapter-04.md)  
* [📚 Литература](./references/chapter-04.md)  
* [💻 Примеры](./examples/chapter-04.md)  
* [🧪 Практика](./exercises/chapter-04.md)

---

## Глава 5. WASM vs WebGPU

Темы:

- Вычисления общего назначения (GPGPU) против рендеринга
- Где заканчиваются вычисления и начинается графика
- Связка WASM + WebGPU как замена нативным движкам
- Управление буферами и шейдерами из WASM

* [📖 Читать главу](./book/chapter-05.md)  
* [📚 Литература](./references/chapter-05.md)  
* [💻 Примеры](./examples/chapter-05.md)  
* [🧪 Практика](./exercises/chapter-05.md)

---

## Глава 6. WASM как второй execution layer Web Platform

Темы:

- Концепция браузерной ОС
- JS для событий, WASM для логики и данных
- Интеграция WASM в Web API
- Эволюция браузеров как мультиязычных сред

* [📖 Читать главу](./book/chapter-06.md)  
* [📚 Литература](./references/chapter-06.md)  
* [💻 Примеры](./examples/chapter-06.md)  
* [🧪 Практика](./exercises/chapter-06.md)

---

### Часть II. WebAssembly Platform

## Глава 7. WASM Runtime: как это работает под капотом

Темы:

- Валидация байт-кода
- Компиляция: AOT, JIT, интерпретация
- Стек вызовов и выполнение инструкций
- Безопасность и изоляция

* [📖 Читать главу](./book/chapter-07.md)  
* [📚 Литература](./references/chapter-07.md)  
* [💻 Примеры](./examples/chapter-07.md)  
* [🧪 Практика](./exercises/chapter-07.md)

---

## Глава 8. Структура WASM Module

Темы:

- Секции модуля: Type, Function, Code, Data, Custom
- Бинарный формат и инструменты (`wasm-objdump`)
- Методы чтения и парсинга
- Кастомизация и метаданные

* [📖 Читать главу](./book/chapter-08.md)  
* [📚 Литература](./references/chapter-08.md)  
* [💻 Примеры](./examples/chapter-08.md)  
* [🧪 Практика](./exercises/chapter-08.md)

---

## Глава 9. Линейная память (Linear Memory)

Темы:

- Модель песочницы
- Работа с сырыми байтами
- Аллокаторы внутри WASM (`wee_alloc`)
- Страницы памяти (64KB) и рост памяти
- Shared Memory

* [📖 Читать главу](./book/chapter-09.md)  
* [📚 Литература](./references/chapter-09.md)  
* [💻 Примеры](./examples/chapter-09.md)  
* [🧪 Практика](./exercises/chapter-09.md)

---

## Глава 10. Система типов WASM

Темы:

- Базовые типы: i32, i64, f32, f64
- Reference Types: externref, anyref
- GC-типы (Garbage Collection) в 2026
- Управление сложными структурами данных

* [📖 Читать главу](./book/chapter-10.md)  
* [📚 Литература](./references/chapter-10.md)  
* [💻 Примеры](./examples/chapter-10.md)  
* [🧪 Практика](./exercises/chapter-10.md)

---

## Глава 11. Imports и Exports

Темы:

- Передача функций из окружения в модуль
- Экспорт функций из модуля
- Экспорт памяти и глобальных переменных
- Таблицы функций (Table)

* [📖 Читать главу](./book/chapter-11.md)  
* [📚 Литература](./references/chapter-11.md)  
* [💻 Примеры](./examples/chapter-11.md)  
* [🧪 Практика](./exercises/chapter-11.md)

---

## Глава 12. WASI (WebAssembly System Interface)

Темы:

- Стандартизация системных вызовов
- Файловая система, время, сеть, случайные числа
- WASI Preview 3 и async I/O
- Запуск WASM за пределами браузера

* [📖 Читать главу](./book/chapter-12.md)  
* [📚 Литература](./references/chapter-12.md)  
* [💻 Примеры](./examples/chapter-12.md)  
* [🧪 Практика](./exercises/chapter-12.md)

---

## Глава 13. Component Model

Темы:

- Интерфейсы (WIT)
- Композиция модулей
- Сборка приложений как LEGO из разных языков
- Примеры композиции

* [📖 Читать главу](./book/chapter-13.md)  
* [📚 Литература](./references/chapter-13.md)  
* [💻 Примеры](./examples/chapter-13.md)  
* [🧪 Практика](./exercises/chapter-13.md)

---

### Часть III. Языки

## Глава 14. Rust → WebAssembly

Темы:

- `wasm-bindgen` и стратегии работы с DOM
- Отсутствие сборщика мусора
- `web-sys` и `js-sys`
- Паника и обработка ошибок

* [📖 Читать главу](./book/chapter-14.md)  
* [📚 Литература](./references/chapter-14.md)  
* [💻 Примеры](./examples/chapter-14.md)  
* [🧪 Практика](./exercises/chapter-14.md)

---

## Глава 15. C/C++ → WebAssembly

Темы:

- Emscripten и его возможности
- Портирование игровых движков
- Эмуляция POSIX
- Управление памятью в C/C++

* [📖 Читать главу](./book/chapter-15.md)  
* [📚 Литература](./references/chapter-15.md)  
* [💻 Примеры](./examples/chapter-15.md)  
* [🧪 Практика](./exercises/chapter-15.md)

---

## Глава 16. Go → WebAssembly

Темы:

- Проблема горутин и сборщика мусора
- TinyGo vs стандартный компилятор
- Когда Go подходит для WASM (CLI, сервер)
- Ограничения Go в браузере

* [📖 Читать главу](./book/chapter-16.md)  
* [📚 Литература](./references/chapter-16.md)  
* [💻 Примеры](./examples/chapter-16.md)  
* [🧪 Практика](./exercises/chapter-16.md)

---

## Глава 17. AssemblyScript

Темы:

- TypeScript-подобный синтаксис для WASM
- Ограничения и отличия от TypeScript
- Производительность vs нативные языки
- Когда выбирать AssemblyScript

* [📖 Читать главу](./book/chapter-17.md)  
* [📚 Литература](./references/chapter-17.md)  
* [💻 Примеры](./examples/chapter-17.md)  
* [🧪 Практика](./exercises/chapter-17.md)

---

## Глава 18. JavaScript ↔ WebAssembly

Темы:

- Вызов JS-функций из WASM
- Вызов WASM-функций из JS
- Передача сложных объектов без копирования (Zero-copy)
- Типичные паттерны интеропа

* [📖 Читать главу](./book/chapter-18.md)  
* [📚 Литература](./references/chapter-18.md)  
* [💻 Примеры](./examples/chapter-18.md)  
* [🧪 Практика](./exercises/chapter-18.md)

---

### Часть IV. WebAssembly в браузере

## Глава 19. Загрузка WASM модулей

Темы:

- `WebAssembly.instantiateStreaming`
- `WebAssembly.instantiate`
- `WebAssembly.compile`
- Стратегии загрузки

* [📖 Читать главу](./book/chapter-19.md)  
* [📚 Литература](./references/chapter-19.md)  
* [💻 Примеры](./examples/chapter-19.md)  
* [🧪 Практика](./exercises/chapter-19.md)

---

## Глава 20. Инстанцирование

Темы:

- От байт-кода к инстансу
- Синхронное и асинхронное инстанцирование
- Передача импортов
- Множественные инстансы одного модуля

* [📖 Читать главу](./book/chapter-20.md)  
* [📚 Литература](./references/chapter-20.md)  
* [💻 Примеры](./examples/chapter-20.md)  
* [🧪 Практика](./exercises/chapter-20.md)

---

## Глава 21. Потоковая компиляция

Темы:

- Оптимизация времени до интерактивности (TTI)
- `WebAssembly.compileStreaming`
- Работа с Response и Fetch API
- Сравнение производительности

* [📖 Читать главу](./book/chapter-21.md)  
* [📚 Литература](./references/chapter-21.md)  
* [💻 Примеры](./examples/chapter-21.md)  
* [🧪 Практика](./exercises/chapter-21.md)

---

## Глава 22. Управление памятью

Темы:

- `WebAssembly.Memory`
- Страницы памяти (64KB)
- Рост памяти: `grow()`
- Утечки памяти в WASM
- Аллокация и освобождение

* [📖 Читать главу](./book/chapter-22.md)  
* [📚 Литература](./references/chapter-22.md)  
* [💻 Примеры](./examples/chapter-22.md)  
* [🧪 Практика](./exercises/chapter-22.md)

---

## Глава 23. JavaScript Interop

Темы:

- Использование `js-sys` и `web-sys` в Rust
- Прокси и хелперы
- Передача колбэков
- Обработка ошибок на границе

* [📖 Читать главу](./book/chapter-23.md)  
* [📚 Литература](./references/chapter-23.md)  
* [💻 Примеры](./examples/chapter-23.md)  
* [🧪 Практика](./exercises/chapter-23.md)

---

## Глава 24. WebAssembly Threads

Темы:

- Многопоточность в WASM
- SharedArrayBuffer
- Атомарные операции
- Безопасность работы с памятью
- Браузерная поддержка

* [📖 Читать главу](./book/chapter-24.md)  
* [📚 Литература](./references/chapter-24.md)  
* [💻 Примеры](./examples/chapter-24.md)  
* [🧪 Практика](./exercises/chapter-24.md)

---

## Глава 25. SIMD

Темы:

- Single Instruction, Multiple Data
- Векторизация вычислений
- Обработка изображений, звука, ML
- Производительность: benchmarks

* [📖 Читать главу](./book/chapter-25.md)  
* [📚 Литература](./references/chapter-25.md)  
* [💻 Примеры](./examples/chapter-25.md)  
* [🧪 Практика](./exercises/chapter-25.md)

---

## Глава 26. Обработка исключений

Темы:

- Механизм Exception Handling в WASM
- Паника в Rust и исключения в C++
- Проброс ошибок в JavaScript
- Практические стратегии

* [📖 Читать главу](./book/chapter-26.md)  
* [📚 Литература](./references/chapter-26.md)  
* [💻 Примеры](./examples/chapter-26.md)  
* [🧪 Практика](./exercises/chapter-26.md)

---

### Часть V. WASM Application Architecture

## Глава 27. WASM как вычислительный слой

Темы:

- Вынос тяжелых математических функций
- Аудио-синтез, криптография, обработка сигналов
- Архитектурные паттерны

* [📖 Читать главу](./book/chapter-27.md)  
* [📚 Литература](./references/chapter-27.md)  
* [💻 Примеры](./examples/chapter-27.md)  
* [🧪 Практика](./exercises/chapter-27.md)

---

## Глава 28. WASM как слой бизнес-логики

Темы:

- Защита кода от инспектирования
- Переносимость логики между бекендом и фронтендом
- Примеры: валидация, финансовые расчеты

* [📖 Читать главу](./book/chapter-28.md)  
* [📚 Литература](./references/chapter-28.md)  
* [💻 Примеры](./examples/chapter-28.md)  
* [🧪 Практика](./exercises/chapter-28.md)

---

## Глава 29. WASM как UI Execution Layer

Темы:

- Canvas + WASM вместо React/Vue для сложных графиков
- Рендеринг нативных UI-библиотек (Flutter, Slint)
- Декларативный UI на WASM

* [📖 Читать главу](./book/chapter-29.md)  
* [📚 Литература](./references/chapter-29.md)  
* [💻 Примеры](./examples/chapter-29.md)  
* [🧪 Практика](./exercises/chapter-29.md)

---

## Глава 30. WASM + Web Workers

Темы:

- Тяжелые фоновые задачи
- Коммуникация между потоками
- Архитектурные шаблоны

* [📖 Читать главу](./book/chapter-30.md)  
* [📚 Литература](./references/chapter-30.md)  
* [💻 Примеры](./examples/chapter-30.md)  
* [🧪 Практика](./exercises/chapter-30.md)

---

## Глава 31. WASM + WebGPU

Темы:

- Управление буферами вершин
- Шейдеры из WASM
- Игровые движки на WASM + WebGPU

* [📖 Читать главу](./book/chapter-31.md)  
* [📚 Литература](./references/chapter-31.md)  
* [💻 Примеры](./examples/chapter-31.md)  
* [🧪 Практика](./exercises/chapter-31.md)

---

## Глава 32. WASM + Web APIs

Темы:

- Геолокация, камера, Bluetooth
- Проксирование нативных API через JS-слой
- Безопасный доступ к устройствам

* [📖 Читать главу](./book/chapter-32.md)  
* [📚 Литература](./references/chapter-32.md)  
* [💻 Примеры](./examples/chapter-32.md)  
* [🧪 Практика](./exercises/chapter-32.md)

---

### Часть VI. Производительность

## Глава 33. WASM Performance

Темы:

- Сравнение WASM vs JS
- Реальные сценарии: Fibonacci, CRC32, JSON парсинг, Image processing
- Бенчмарки и интерпретация результатов

* [📖 Читать главу](./book/chapter-33.md)  
* [📚 Литература](./references/chapter-33.md)  
* [💻 Примеры](./examples/chapter-33.md)  
* [🧪 Практика](./exercises/chapter-33.md)

---

## Глава 34. Startup Cost

Темы:

- Время компиляции и инстанцирования
- Кэширование скомпилированного кода
- Стратегии снижения стоимости старта

* [📖 Читать главу](./book/chapter-34.md)  
* [📚 Литература](./references/chapter-34.md)  
* [💻 Примеры](./examples/chapter-34.md)  
* [🧪 Практика](./exercises/chapter-34.md)

---

## Глава 35. Memory

Темы:

- Потребление памяти WASM-модулями
- Фрагментация линейной памяти
- Оптимизация аллокаций

* [📖 Читать главу](./book/chapter-35.md)  
* [📚 Литература](./references/chapter-35.md)  
* [💻 Примеры](./examples/chapter-35.md)  
* [🧪 Практика](./exercises/chapter-35.md)

---

## Глава 36. Binary Size

Темы:

- Стратегии оптимизации: LTO, `wasm-opt`
- Измерение эффективности транспорта (gzip/brotli)
- Уменьшение размера бинарного файла

* [📖 Читать главу](./book/chapter-36.md)  
* [📚 Литература](./references/chapter-36.md)  
* [💻 Примеры](./examples/chapter-36.md)  
* [🧪 Практика](./exercises/chapter-36.md)

---

## Глава 37. Profiling

Темы:

- DevTools 2026: анализ Flame Chart
- Инструменты: `chrome://tracing`
- Профайлеры для WASM
- Поиск узких мест

* [📖 Читать главу](./book/chapter-37.md)  
* [📚 Литература](./references/chapter-37.md)  
* [💻 Примеры](./examples/chapter-37.md)  
* [🧪 Практика](./exercises/chapter-37.md)

---

## Глава 38. Benchmarking

Темы:

- Создание бенчмарков для WASM
- Интерпретация результатов
- Сравнение разных языков и подходов

* [📖 Читать главу](./book/chapter-38.md)  
* [📚 Литература](./references/chapter-38.md)  
* [💻 Примеры](./examples/chapter-38.md)  
* [🧪 Практика](./exercises/chapter-38.md)

---

### Часть VII. Production

## Глава 39. WASM Security

Темы:

- Песочница и изоляция
- CORS и политики безопасности
- Уязвимости: переполнение буфера
- Безопасная работа с памятью

* [📖 Читать главу](./book/chapter-39.md)  
* [📚 Литература](./references/chapter-39.md)  
* [💻 Примеры](./examples/chapter-39.md)  
* [🧪 Практика](./exercises/chapter-39.md)

---

## Глава 40. Развертывание (Deployment)

Темы:

- Настройка CDN для `.wasm` файлов
- MIME-типы
- HTTP-заголовки: COOP, CORP
- Стратегии деплоя

* [📖 Читать главу](./book/chapter-40.md)  
* [📚 Литература](./references/chapter-40.md)  
* [💻 Примеры](./examples/chapter-40.md)  
* [🧪 Практика](./exercises/chapter-40.md)

---

## Глава 41. Кэширование

Темы:

- Кэширование скомпилированного кода
- Cache API
- Версионирование модулей

* [📖 Читать главу](./book/chapter-41.md)  
* [📚 Литература](./references/chapter-41.md)  
* [💻 Примеры](./examples/chapter-41.md)  
* [🧪 Практика](./exercises/chapter-41.md)

---

## Глава 42. CDN

Темы:

- Оптимизация доставки бинарных файлов
- Edge-кэширование
- Стратегии сжатия

* [📖 Читать главу](./book/chapter-42.md)  
* [📚 Литература](./references/chapter-42.md)  
* [💻 Примеры](./examples/chapter-42.md)  
* [🧪 Практика](./exercises/chapter-42.md)

---

## Глава 43. Версионирование

Темы:

- Управление версиями WASM-модулей
- Совместимость и обратная совместимость
- Стратегии миграции

* [📖 Читать главу](./book/chapter-43.md)  
* [📚 Литература](./references/chapter-43.md)  
* [💻 Примеры](./examples/chapter-43.md)  
* [🧪 Практика](./exercises/chapter-43.md)

---

## Глава 44. Отладка в Production

Темы:

- Source Maps для WASM (DWARF)
- Логирование и мониторинг
- Sentry и аналоги для WASM-паники

* [📖 Читать главу](./book/chapter-44.md)  
* [📚 Литература](./references/chapter-44.md)  
* [💻 Примеры](./examples/chapter-44.md)  
* [🧪 Практика](./exercises/chapter-44.md)

---

### Часть VIII. WebAssembly 2030

## Глава 45. Зрелость Component Model

Темы:

- Эра композиции
- Сборка систем из компонентов
- Языконезависимые интерфейсы (WIT)

* [📖 Читать главу](./book/chapter-45.md)  
* [📚 Литература](./references/chapter-45.md)  
* [💻 Примеры](./examples/chapter-45.md)  
* [🧪 Практика](./exercises/chapter-45.md)

---

## Глава 46. WASI как операционная система

Темы:

- WASI как новая абстракция ОС
- Запуск WASM на сервере, IoT, десктопе
- Wasmtime, Wasmer и другие рантаймы

* [📖 Читать главу](./book/chapter-46.md)  
* [📚 Литература](./references/chapter-46.md)  
* [💻 Примеры](./examples/chapter-46.md)  
* [🧪 Практика](./exercises/chapter-46.md)

---

## Глава 47. Server-side WASM

Темы:

- WASM как альтернатива контейнерам
- Безопасность и скорость старта
- Платформы: Fermyon Spin, Fastly Compute

* [📖 Читать главу](./book/chapter-47.md)  
* [📚 Литература](./references/chapter-47.md)  
* [💻 Примеры](./examples/chapter-47.md)  
* [🧪 Практика](./exercises/chapter-47.md)

---

## Глава 48. Edge WASM

Темы:

- Почему WASM выигрывает на периферии
- Низкий старт и малый размер
- Serverless WASM

* [📖 Читать главу](./book/chapter-48.md)  
* [📚 Литература](./references/chapter-48.md)  
* [💻 Примеры](./examples/chapter-48.md)  
* [🧪 Практика](./exercises/chapter-48.md)

---

## Глава 49. Универсальные компоненты

Темы:

- Один бинарник везде
- От микроконтроллера до облака
- Переносимость и будущее разработки

* [📖 Читать главу](./book/chapter-49.md)  
* [📚 Литература](./references/chapter-49.md)  
* [💻 Примеры](./examples/chapter-49.md)  
* [🧪 Практика](./exercises/chapter-49.md)

---

## Глава 50. Будущее WebAssembly

Темы:

- Уход от JavaScript в корне?
- Нативная поддержка WASM GC
- DOM-манипуляции через WASM-биндинги
- Дорожная карта WebAssembly

* [📖 Читать главу](./book/chapter-50.md)  
* [📚 Литература](./references/chapter-50.md)  
* [💻 Примеры](./examples/chapter-50.md)  
* [🧪 Практика](./exercises/chapter-50.md)

---

### Заключение

## Новая эра Web Platform

- WebAssembly не заменяет JavaScript
- Дополняет, превращая веб в высокопроизводительную среду
- Многопоточность, безопасность, переносимость
- Приложения любого масштаба

* [📖 Читать главу](./book/conclusion.md)  
* [📚 Литература](./references/conclusion.md)  
* [💻 Примеры](./examples/conclusion.md)

---

### Приложения

## Приложение A. Справочник по инструкциям WASM

- Полный список Opcode
- Описание инструкций
- Примеры использования

* [📖 Читать главу](./book/appendix-a.md)

---

## Приложение B. Настройка окружения

- `wasm-tools`
- `wat2wasm`
- `cargo-wasi`
- Сборка проекта

* [📖 Читать главу](./book/appendix-b.md)

---

## Приложение C. Глоссарий терминов

- Linear Memory
- Traps
- Stacks
- WIT
- Component Model

* [📖 Читать главу](./book/appendix-c.md)

---

## Приложение D. Код-примеры основных паттернов

- Rust + WASM
- C++ + Emscripten
- AssemblyScript
- JavaScript Interop

* [📖 Читать главу](./book/appendix-d.md)

---

## 🤝 Как помочь

Вы можете внести свой вклад в развитие книги:

- 🐛 Сообщить об ошибке в Issues
- ✏️ Предложить правку через Pull Request
- 💡 Подсказать тему для новой главы
- 🌍 Помочь с переводом на другие языки

---

## 📄 Лицензия

Эта книга распространяется под лицензией [Creative Commons Attribution-ShareAlike 4.0 International (CC BY-SA 4.0)](https://creativecommons.org/licenses/by-sa/4.0/).

Вы можете свободно:

- **Делиться** — копировать и распространять материал на любом носителе и в любом формате;
- **Адаптировать** — изменять и дополнять материал для любых целей.

При условии соблюдения:

- **Атрибуция** — вы должны указать авторство;
- **ShareAlike** — если вы изменяете материал, вы должны распространять его на тех же условиях.

---

**«WebAssembly as a Modern Web Platform 2026»** — ваш путеводитель в мир высокопроизводительных вычислений в вебе.
