# C# .NET 5+ — Шпаргалка: Async, Parallel, Синхронизация

Практический справочник для повседневного кодинга. Каждый раздел: **что это**, **когда использовать**, **пример**, **грабли**.

---

## Оглавление

1. [Ключевые принципы (прочитать первым)](#1-ключевые-принципы-прочитать-первым)
2. [Async/Await — основы](#2-asyncawait--основы)
3. [Task, ValueTask, Task<T>](#3-task-valuetask-taskt)
4. [Композиция задач: WhenAll, WhenAny, WaitAsync](#4-композиция-задач-whenall-whenany-waitasync)
5. [Отмена операций: CancellationToken](#5-отмена-операций-cancellationtoken)
6. [Таймауты и Progress](#6-таймауты-и-progress)
7. [ConfigureAwait и контекст синхронизации](#7-configureawait-и-контекст-синхронизации)
8. [IAsyncEnumerable и потоковая асинхронность](#8-iasyncenumerable-и-потоковая-асинхронность)
9. [Обработка исключений в async-коде](#9-обработка-исключений-в-async-коде)
10. [Параллельное программирование (CPU-bound)](#10-параллельное-программирование-cpu-bound)
11. [PLINQ](#11-plinq)
12. [Синхронизация потоков: примитивы](#12-синхронизация-потоков-примитивы)
13. [Lock-free: Interlocked и volatile](#13-lock-free-interlocked-и-volatile)
14. [Потокобезопасные коллекции](#14-потокобезопасные-коллекции)
15. [System.Threading.Channels](#15-systemthreadingchannels)
16. [Producer/Consumer и координация](#16-producerconsumer-и-координация)
17. [Dataflow (TPL Dataflow)](#17-dataflow-tpl-dataflow)
18. [Асинхронная инициализация и Lazy](#18-асинхронная-инициализация-и-lazy)
19. [Тайминги, троттлинг, ретраи](#19-тайминги-троттлинг-ретраи)
20. [Типичные ошибки и антипаттерны](#20-типичные-ошибки-и-антипаттерны)
21. [Диагностика и отладка](#21-диагностика-и-отладка)
22. [Выбор инструмента: таблица решений](#22-выбор-инструмента-таблица-решений)
23. [Чек-лист перед коммитом](#23-чек-лист-перед-коммитом)

---

## 1. Ключевые принципы (прочитать первым)

- **Async ≠ Parallel.** `async/await` — про неблокирующее ожидание I/O (сеть, диск, БД). Parallel (`Task.Run`, `Parallel.For`, PLINQ) — про распараллеливание CPU-bound вычислений на несколько ядер. Не путать: `Task.Run(() => await SomethingAsync())` почти всегда бессмысленно.
- **"Async всё до самого верха" (async all the way).** Не смешивайте блокирующие вызовы (`.Result`, `.Wait()`, `GetAwaiter().GetResult()`) с асинхронным кодом — это главный источник дедлоков.
- **I/O-bound → async/await.** CPU-bound → `Task.Run` / `Parallel` / PLINQ.
- **Дешёвое ожидание.** `await` не блокирует поток — он освобождает его в пул, пока идёт I/O. Это главное преимущество async для масштабируемости серверов.
- **Task — это "горячая" (уже запущенная) операция**, а не отложенное действие. В отличие от `IEnumerable`/`Func`, `Task` начинает выполняться сразу при создании (кроме `Task` в состоянии `Created`, что используется редко).
- **Иммутабельность и минимизация shared state** — лучшая стратегия синхронизации: если нечего делить между потоками, не нужны и блокировки.

---

## 2. Async/Await — основы

**Когда:** любой I/O-bound код — HTTP-запросы, работа с БД, файлами, сокетами.

```csharp
public async Task<string> GetDataAsync(HttpClient client, string url)
{
    // await освобождает поток на время ожидания ответа
    HttpResponseMessage response = await client.GetAsync(url);
    response.EnsureSuccessStatusCode();
    return await response.Content.ReadAsStringAsync();
}
```

**Правила именования и сигнатур:**
- Суффикс `Async` для всех асинхронных методов: `GetDataAsync`, `SaveAsync`.
- Возвращайте `Task`, `Task<T>`, `ValueTask`, `ValueTask<T>` или `void` — только для обработчиков событий.
- Избегайте `async void` — исключения из него нельзя перехватить снаружи, они падают прямо в `SynchronizationContext`/`AppDomain` и роняют процесс.

```csharp
// ПЛОХО — async void, кроме обработчиков событий
async void ButtonClick(object sender, EventArgs e) { ... } // OK — единственное исключение

// ПЛОХО в обычном коде
public async void DoWork() => await SomethingAsync(); // нельзя await, нельзя поймать исключение

// ХОРОШО
public async Task DoWorkAsync() => await SomethingAsync();
```

---

## 3. Task, ValueTask, Task<T>

| Тип | Когда использовать |
|---|---|
| `Task` | Асинхронная операция без результата |
| `Task<T>` | Асинхронная операция, возвращающая значение |
| `ValueTask` / `ValueTask<T>` | Горячий путь с частым синхронным (кэшированным) завершением — экономия аллокаций |
| `IAsyncEnumerable<T>` | Асинхронный поток из нескольких значений |

```csharp
// ValueTask — оправдан, когда результат часто уже готов (кэш и т.п.)
public ValueTask<int> GetCachedOrLoadAsync(string key)
{
    if (_cache.TryGetValue(key, out var value))
        return new ValueTask<int>(value); // без аллокации Task

    return new ValueTask<int>(LoadFromDbAsync(key));
}
```

**Правила ValueTask (важно!):**
- Нельзя `await` дважды один и тот же `ValueTask`.
- Нельзя вызывать `.Result`/`.GetAwaiter().GetResult()` до завершения.
- Нельзя одновременно ожидать из нескольких мест (в отличие от `Task`).
- Если сомневаетесь — используйте `Task<T>`. `ValueTask` — оптимизация для горячих путей с профилированием, не default-выбор.

**Создание завершённых задач без накладных расходов:**

```csharp
public Task<int> GetDefaultAsync() => Task.FromResult(42);
public static readonly Task CompletedTask = Task.CompletedTask; // без результата
```

---

## 4. Композиция задач: WhenAll, WhenAny, WaitAsync

**`Task.WhenAll`** — запустить несколько независимых операций параллельно и дождаться всех:

```csharp
Task<string> t1 = GetDataAsync(url1);
Task<string> t2 = GetDataAsync(url2);
Task<string> t3 = GetDataAsync(url3);

string[] results = await Task.WhenAll(t1, t2, t3);
```

**Грабли WhenAll:** если несколько задач бросают исключения, `await Task.WhenAll` пробрасывает только первое. Остальные видны через `task.Exception`:

```csharp
var all = Task.WhenAll(t1, t2, t3);
try { await all; }
catch
{
    var errors = all.Exception?.InnerExceptions ?? Enumerable.Empty<Exception>();
    foreach (var ex in errors) Log(ex);
}
```

**`Task.WhenAny`** — гонка задач (первый ответивший сервер, таймаут):

```csharp
var timeoutTask = Task.Delay(TimeSpan.FromSeconds(5));
var workTask = DoWorkAsync();

var finished = await Task.WhenAny(workTask, timeoutTask);
if (finished == timeoutTask)
    throw new TimeoutException();

var result = await workTask; // безопасно, уже завершена
```

**`.WaitAsync(timeout/token)` (.NET 6+)** — компактный способ добавить таймаут/отмену к любой задаче:

```csharp
await SomeOperationAsync().WaitAsync(TimeSpan.FromSeconds(10), cancellationToken);
```

---

## 5. Отмена операций: CancellationToken

**Когда:** любая долгая или потенциально бесконечная операция должна поддерживать отмену.

```csharp
public async Task ProcessAsync(CancellationToken ct = default)
{
    for (int i = 0; i < items.Count; i++)
    {
        ct.ThrowIfCancellationRequested(); // ручная проверка в CPU-циклах
        await ProcessItemAsync(items[i], ct); // проброс токена дальше
    }
}
```

**Источник отмены и таймаут:**

```csharp
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(30));
using var linked = CancellationTokenSource.CreateLinkedTokenSource(cts.Token, externalToken);

try
{
    await ProcessAsync(linked.Token);
}
catch (OperationCanceledException) when (cts.IsCancellationRequested)
{
    // именно наш таймаут, а не внешняя отмена
}
```

**Правила:**
- Всегда принимайте `CancellationToken` параметром (с `= default`) в публичных async-методах.
- Всегда пробрасывайте токен во все вложенные async-вызовы.
- Ловите `OperationCanceledException` (или его наследник `TaskCanceledException`), а не общий `Exception`.
- `CancellationTokenSource` реализует `IDisposable` — оборачивайте в `using`.
- Не путайте отмену с ошибкой: отмена — это штатный поток управления, не логируйте её как ошибку.

---

## 6. Таймауты и Progress

```csharp
// Таймаут через CancellationTokenSource
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
await LongRunningAsync(cts.Token);

// Прогресс из фоновой задачи в UI/лог
var progress = new Progress<int>(percent => Console.WriteLine($"{percent}%"));
await DownloadAsync(url, progress);

async Task DownloadAsync(string url, IProgress<int> progress)
{
    for (int i = 0; i <= 100; i += 10)
    {
        await Task.Delay(100);
        progress.Report(i); // безопасно вызывается из любого потока,
                             // callback выполнится в контексте, где создан Progress<T>
    }
}
```

---

## 7. ConfigureAwait и контекст синхронизации

**Что это:** после `await` выполнение по умолчанию возвращается в исходный `SynchronizationContext` (UI-поток в WPF/WinForms) или `TaskScheduler`.

**Правило:**
- **Библиотечный/доменный код (не касается UI)** → `ConfigureAwait(false)` на каждом `await`, чтобы не тащить UI-контекст и избежать дедлоков.
- **Код в ASP.NET Core** → `SynchronizationContext` отсутствует (начиная с .NET Core), `ConfigureAwait(false)` не обязателен, но не вреден.
- **UI-код (WPF/WinForms/MAUI), где после await нужно трогать UI** → без `ConfigureAwait(false)` (либо `ConfigureAwait(true)`, значение по умолчанию).

```csharp
// Библиотека — не знаем и не должны знать, кто её вызывает
public async Task<Data> LoadAsync()
{
    var raw = await _client.GetStringAsync(url).ConfigureAwait(false);
    return Parse(raw);
}
```

**.NET 6+ альтернатива на уровне метода:** атрибут `[module: SkipLocalsInit]`-подобного рода не для этого; вместо расстановки `ConfigureAwait(false)` везде используйте анализатор `CA2007` (включён в `.editorconfig`) или в .NET 8+ рассмотрите использование `ConfigureAwaitOptions`:

```csharp
await task.ConfigureAwait(ConfigureAwaitOptions.ContinueOnCapturedContext | ConfigureAwaitOptions.SuppressThrowing);
```

---

## 8. IAsyncEnumerable и потоковая асинхронность

**Когда:** нужно отдавать элементы по одному по мере готовности (пагинация API, чтение большого файла/стрима, работа с БД курсорами).

```csharp
public async IAsyncEnumerable<Order> GetOrdersAsync(
    [EnumeratorCancellation] CancellationToken ct = default)
{
    await foreach (var page in FetchPagesAsync(ct))
    {
        foreach (var order in page.Items)
            yield return order;
    }
}

// Потребление
await foreach (var order in GetOrdersAsync(ct))
{
    Console.WriteLine(order.Id);
}
```

**Грабли:** обязательно ставьте атрибут `[EnumeratorCancellation]` на параметр токена — иначе он не передастся внутрь генератора корректно.

---

## 9. Обработка исключений в async-коде

```csharp
public async Task<Result> SafeCallAsync()
{
    try
    {
        return await RiskyAsync();
    }
    catch (HttpRequestException ex)
    {
        Log(ex);
        throw; // сохраняет stack trace, не new Exception(...)
    }
}
```

**Правила:**
- Исключение из `async Task` метода "материализуется" при `await`, а не в момент вызова — try/catch вокруг вызова без `await` его не поймает.
- `AggregateException` возникает только при синхронном `.Wait()`/`.Result`. При `await` исключение "разворачивается" в первое исходное.
- Для нескольких задач и всех их исключений — смотрите `Task.Exception.InnerExceptions` (раздел 4) или `Task.WhenAll` с ручным сбором.

```csharp
// ПЛОХО: исключение не поймается здесь — оно "в будущем", при await
try
{
    var task = RiskyAsync(); // синхронный вызов, ещё не await
}
catch { } // не сработает для ошибки внутри RiskyAsync
```

---

## 10. Параллельное программирование (CPU-bound)

**Когда:** тяжёлые вычисления, которые можно разбить на независимые куски и распределить по ядрам CPU. НЕ для I/O — там нужен async/await.

### Task.Run — разовая работа в пуле потоков

```csharp
int result = await Task.Run(() => HeavyComputation(data));
```

### Parallel.For / Parallel.ForEach

```csharp
Parallel.For(0, items.Count, i =>
{
    Process(items[i]);
});

Parallel.ForEach(items, item => Process(item));

// С ограничением степени параллелизма и отменой
var options = new ParallelOptions
{
    MaxDegreeOfParallelism = Environment.ProcessorCount,
    CancellationToken = ct
};
Parallel.ForEach(items, options, item => Process(item));
```

### Parallel.ForEachAsync (.NET 6+) — CPU + асинхронная работа внутри цикла

```csharp
await Parallel.ForEachAsync(urls, new ParallelOptions
{
    MaxDegreeOfParallelism = 8,
    CancellationToken = ct
}, async (url, token) =>
{
    var data = await client.GetStringAsync(url, token);
    Process(data);
});
```

### Parallel.Invoke — набор независимых действий

```csharp
Parallel.Invoke(
    () => Task1(),
    () => Task2(),
    () => Task3()
);
```

**Грабли параллелизма:**
- Не распараллеливайте мелкие/дешёвые операции — накладные расходы на планирование превысят выгоду.
- Не изменяйте общее состояние (списки, словари) без синхронизации внутри тела `Parallel.For` — используйте `ConcurrentBag`, `ConcurrentDictionary` либо локальные накопители (`Parallel.For` с `localInit`/`localFinally`).
- `MaxDegreeOfParallelism` — ставьте явно, если не хотите занять весь пул потоков (особенно на сервере под нагрузкой).

```csharp
// Аккумуляция результатов через thread-local storage — без блокировок
long total = 0;
Parallel.For(0, data.Length,
    localInit: () => 0L,
    body: (i, state, localSum) => localSum + data[i],
    localFinally: localSum => Interlocked.Add(ref total, localSum));
```

---

## 11. PLINQ

**Когда:** декларативная обработка больших коллекций in-memory с автоматическим распараллеливанием LINQ-запроса.

```csharp
var results = data.AsParallel()
    .WithDegreeOfParallelism(Environment.ProcessorCount)
    .WithCancellation(ct)
    .Where(x => IsExpensive(x))
    .Select(x => Transform(x))
    .ToList();

// Сохранение порядка (дороже по производительности)
var ordered = data.AsParallel().AsOrdered().Select(Transform).ToList();

// ForAll — обработка без сборки результата (быстрее, чем ToList + foreach)
data.AsParallel().ForAll(x => Process(x));
```

**Когда НЕ использовать PLINQ:**
- Маленькие коллекции (< нескольких тысяч элементов) — оверхед разбиения на партиции не окупится.
- Операции с I/O внутри запроса — используйте async-подходы, а не PLINQ.
- Побочные эффекты в лямбдах, зависящие от порядка выполнения.

---

## 12. Синхронизация потоков: примитивы

| Примитив | Область | Асинхронно? | Когда |
|---|---|---|---|
| `lock` (`Monitor`) | Внутри процесса | Нет | Простая взаимоисключающая секция, синхронный код |
| `SemaphoreSlim` | Внутри процесса | Да (`WaitAsync`) | Ограничение параллелизма, в т.ч. в async-коде |
| `Mutex` | Между процессами | Нет | Межпроцессная синхронизация (например, "только один экземпляр приложения") |
| `ReaderWriterLockSlim` | Внутри процесса | Нет | Много читателей, редкие писатели |
| `Monitor.Wait/Pulse` | Внутри процесса | Нет | Классический producer/consumer вручную (обычно лучше `Channel`) |
| `Barrier` | Внутри процесса | Нет | Синхронизация фаз в группе потоков |
| `CountdownEvent` | Внутри процесса | Нет | Дождаться N завершений |
| `ManualResetEventSlim` / `AutoResetEvent` | Внутри процесса | Нет | Сигнализация между потоками |

### lock / Monitor

```csharp
private readonly object _sync = new();
private int _counter;

public void Increment()
{
    lock (_sync)
    {
        _counter++;
    }
}
```

**Правила:**
- Объект блокировки — приватное, выделенное поле (`private readonly object _sync = new()`), никогда не `this`, не публичный объект, не строка (интернирование строк!).
- **Никогда не используйте `await` внутри `lock`** — компилятор это запретит для `async` методов, и правильно: `lock` держит поток, а `await` может его освободить/сменить, что ломает семантику блокировки. Используйте `SemaphoreSlim` вместо `lock` в асинхронном коде.

```csharp
// ОШИБКА КОМПИЛЯЦИИ — так и должно быть
lock (_sync)
{
    await SomethingAsync(); // CS1996: await не разрешён в теле lock
}
```

### SemaphoreSlim — асинхронная замена lock

```csharp
private readonly SemaphoreSlim _semaphore = new(1, 1); // как async-lock

public async Task UpdateAsync()
{
    await _semaphore.WaitAsync();
    try
    {
        await DoWorkAsync();
    }
    finally
    {
        _semaphore.Release();
    }
}
```

**Ограничение параллелизма (например, не больше 5 одновременных HTTP-запросов):**

```csharp
private readonly SemaphoreSlim _throttle = new(initialCount: 5, maxCount: 5);

public async Task<string> FetchAsync(string url)
{
    await _throttle.WaitAsync();
    try { return await _client.GetStringAsync(url); }
    finally { _throttle.Release(); }
}
```

### ReaderWriterLockSlim

```csharp
private readonly ReaderWriterLockSlim _rwLock = new();
private Dictionary<string, string> _cache = new();

public string? Read(string key)
{
    _rwLock.EnterReadLock();
    try { return _cache.GetValueOrDefault(key); }
    finally { _rwLock.ExitReadLock(); }
}

public void Write(string key, string value)
{
    _rwLock.EnterWriteLock();
    try { _cache[key] = value; }
    finally { _rwLock.ExitWriteLock(); }
}
```

Используйте, когда чтений заметно больше, чем записей; при частых записях выигрыш относительно обычного `lock` теряется.

### Mutex — межпроцессная блокировка

```csharp
using var mutex = new Mutex(initiallyOwned: false, name: @"Global\MyAppSingleInstance");
if (!mutex.WaitOne(TimeSpan.Zero))
{
    Console.WriteLine("Уже запущено");
    return;
}
try { RunApp(); }
finally { mutex.ReleaseMutex(); }
```

---

## 13. Lock-free: Interlocked и volatile

**Когда:** простые атомарные операции (счётчики, флаги) без накладных расходов блокировки.

```csharp
private int _counter;
public void Increment() => Interlocked.Increment(ref _counter);
public void Add(int value) => Interlocked.Add(ref _counter, value);

// Атомарная замена с проверкой (CAS)
private int _state;
public bool TryActivate() =>
    Interlocked.CompareExchange(ref _state, 1, 0) == 0;

// Атомарное чтение/запись ссылочного типа
private volatile bool _isRunning;
public void Stop() => _isRunning = false;
```

**Правила:**
- `Interlocked` — для одиночных атомарных операций над `int`/`long`/`object` ссылками. Не заменяет `lock` для составных операций ("прочитать-изменить-записать несколько полей").
- `volatile` — гарантирует видимость изменений между потоками без переупорядочивания компилятором/CPU, но **не делает операцию атомарной** (`volatile int x; x++;` всё равно не потокобезопасно). Используйте для простых флагов остановки.
- Не изобретайте свои lock-free структуры данных без крайней необходимости — используйте готовые `System.Collections.Concurrent`.

---

## 14. Потокобезопасные коллекции

| Коллекция | Замена | Особенности |
|---|---|---|
| `ConcurrentDictionary<K,V>` | `Dictionary` | `GetOrAdd`, `AddOrUpdate` — атомарны, но делегат внутри может выполниться повторно |
| `ConcurrentQueue<T>` | `Queue` | FIFO, lock-free |
| `ConcurrentStack<T>` | `Stack` | LIFO, lock-free |
| `ConcurrentBag<T>` | — | Неупорядоченная, оптимизирована под сценарий "каждый поток пишет и читает своё" |
| `BlockingCollection<T>` | — | Producer/consumer с блокировкой при пустой/полной очереди, поддерживает `CompleteAdding` |

```csharp
var dict = new ConcurrentDictionary<string, int>();
dict.AddOrUpdate("key", 1, (k, old) => old + 1);

// GetOrAdd: фабрика может вызваться параллельно несколько раз — не должна иметь побочных эффектов
var value = dict.GetOrAdd("key", k => ExpensiveCompute(k));
```

**Правило:** предпочитайте `ConcurrentDictionary`/`Channel` собственноручной блокировке обычного `Dictionary` — меньше кода, меньше шансов на ошибку.

---

## 15. System.Threading.Channels

**Когда:** современная замена `BlockingCollection` для producer/consumer в async-коде. Стандарт де-факто с .NET Core 3+ / .NET 5+.

```csharp
var channel = Channel.CreateBounded<int>(new BoundedChannelOptions(capacity: 100)
{
    FullMode = BoundedChannelFullMode.Wait,
    SingleReader = false,
    SingleWriter = false
});

// Producer
async Task ProduceAsync(CancellationToken ct)
{
    for (int i = 0; i < 1000; i++)
        await channel.Writer.WriteAsync(i, ct);

    channel.Writer.Complete(); // сигнал об окончании
}

// Consumer
async Task ConsumeAsync(CancellationToken ct)
{
    await foreach (var item in channel.Reader.ReadAllAsync(ct))
    {
        Process(item);
    }
}

await Task.WhenAll(ProduceAsync(ct), ConsumeAsync(ct));
```

**`Channel.CreateUnbounded<T>()`** — без ограничения по размеру (осторожно с ростом памяти).
**`BoundedChannelFullMode`** — `Wait` (по умолчанию, backpressure), `DropOldest`, `DropNewest`, `DropWrite`.

---

## 16. Producer/Consumer и координация

```csharp
// BlockingCollection — синхронный producer/consumer (legacy, но иногда уместен)
var queue = new BlockingCollection<int>(boundedCapacity: 50);

var producer = Task.Run(() =>
{
    for (int i = 0; i < 100; i++) queue.Add(i);
    queue.CompleteAdding();
});

var consumer = Task.Run(() =>
{
    foreach (var item in queue.GetConsumingEnumerable())
        Process(item);
});

Task.WaitAll(producer, consumer);
```

**Рекомендация:** для нового кода в async-контексте — `Channel<T>` (раздел 15). `BlockingCollection` — для чисто синхронных worker-потоков.

---

## 17. Dataflow (TPL Dataflow)

**Когда:** построение конвейеров обработки (pipeline) с несколькими стадиями, каждая со своей степенью параллелизма. Пакет `System.Threading.Tasks.Dataflow`.

```csharp
var downloadBlock = new TransformBlock<string, string>(
    async url => await client.GetStringAsync(url),
    new ExecutionDataflowBlockOptions { MaxDegreeOfParallelism = 4 });

var parseBlock = new TransformBlock<string, Data>(html => Parse(html));

var saveBlock = new ActionBlock<Data>(
    async data => await SaveAsync(data),
    new ExecutionDataflowBlockOptions { MaxDegreeOfParallelism = 2 });

var linkOptions = new DataflowLinkOptions { PropagateCompletion = true };
downloadBlock.LinkTo(parseBlock, linkOptions);
parseBlock.LinkTo(saveBlock, linkOptions);

foreach (var url in urls) await downloadBlock.SendAsync(url);
downloadBlock.Complete();

await saveBlock.Completion;
```

Используйте, когда нужен многостадийный конвейер с разной степенью параллелизма на каждом шаге. Для простого fan-out/fan-in обычно достаточно `Channel` + несколько `Task`.

---

## 18. Асинхронная инициализация и Lazy

```csharp
// Ленивая, потокобезопасная, однократная асинхронная инициализация
private readonly Lazy<Task<Config>> _config =
    new(() => LoadConfigAsync(), LazyThreadSafetyMode.ExecutionAndPublication);

public Task<Config> GetConfigAsync() => _config.Value;
```

**Грабли:** без `Lazy<Task<T>>` несколько потоков могут одновременно запустить дорогую инициализацию. `Lazy<T>` гарантирует единственный запуск фабрики при `ExecutionAndPublication` (по умолчанию).

---

## 19. Тайминги, троттлинг, ретраи

```csharp
// Task.Delay вместо Thread.Sleep в async-коде — не блокирует поток
await Task.Delay(TimeSpan.FromSeconds(2), ct);

// Периодическая работа — PeriodicTimer (.NET 6+), предпочтительнее System.Timers.Timer
using var timer = new PeriodicTimer(TimeSpan.FromSeconds(30));
while (await timer.WaitForNextTickAsync(ct))
{
    await DoPeriodicWorkAsync(ct);
}
```

**Ретраи с экспоненциальной задержкой (вручную; в проде — рассмотрите Polly):**

```csharp
async Task<T> RetryAsync<T>(Func<Task<T>> action, int maxAttempts = 3)
{
    for (int attempt = 1; ; attempt++)
    {
        try { return await action(); }
        catch (Exception) when (attempt < maxAttempts)
        {
            await Task.Delay(TimeSpan.FromMilliseconds(200 * Math.Pow(2, attempt)));
        }
    }
}
```

**Правило:** никогда `Thread.Sleep` внутри `async`-метода — это блокирует поток пула. Всегда `await Task.Delay(...)`.

---

## 20. Типичные ошибки и антипаттерны

### 20.1 Блокирующее ожидание async-кода (главная причина дедлоков)

```csharp
// ПЛОХО — классический дедлок в UI/ASP.NET (legacy) с SynchronizationContext
public string GetData()
{
    return GetDataAsync().Result; // или .Wait(), .GetAwaiter().GetResult()
}
```
Поток вызывающего блокируется в ожидании `Result`, но продолжение `GetDataAsync` после `await` пытается вернуться в тот же (занятый) синхронизирующий контекст → дедлок.

**Решение:** делайте вызывающий код тоже `async` до самого верха. Если совсем нельзя (например, `Main` до C# 7.1, или интерфейс библиотеки синхронный) — минимум используйте `.ConfigureAwait(false)` во всей цепочке вызовов и/или `Task.Run(() => AsyncMethod()).GetAwaiter().GetResult()` как компромисс.

### 20.2 async void вне обработчиков событий — см. раздел 2.

### 20.3 Task.Run для I/O-bound кода

```csharp
// ПЛОХО — тратит поток пула впустую на ожидание I/O
await Task.Run(() => client.GetStringAsync(url).Result);

// ХОРОШО
await client.GetStringAsync(url);
```

### 20.4 Забытый await ("fire and forget" по ошибке)

```csharp
// ПЛОХО — исключение потеряется, метод продолжится не дождавшись
public async Task ProcessAsync()
{
    SaveAsync(data); // забыли await!
    Log("done"); // выполнится раньше, чем реально сохранится
}
```
Компилятор выдаёт warning CS4014 — не игнорируйте его.

**Осознанный fire-and-forget** — явно оборачивайте и логируйте исключения:

```csharp
_ = Task.Run(async () =>
{
    try { await BackgroundJobAsync(); }
    catch (Exception ex) { Log(ex); }
});
```

### 20.5 Захват `foreach`-переменной / состояния гонки в замыканиях

```csharp
// В C# 5+ переменная цикла foreach уже per-iteration — это безопасно.
// Но для for/while с общей переменной — осторожно:
var tasks = new List<Task>();
for (int i = 0; i < 5; i++)
{
    int captured = i; // явный захват — обязателен для for/while
    tasks.Add(Task.Run(() => Console.WriteLine(captured)));
}
await Task.WhenAll(tasks);
```

### 20.6 Modifying коллекцию во время параллельной итерации

```csharp
// ПЛОХО — InvalidOperationException или повреждение данных
Parallel.ForEach(list, item => list.Remove(item));

// ХОРОШО — собрать результат отдельно, затем применить
var toRemove = new ConcurrentBag<Item>();
Parallel.ForEach(list, item => { if (ShouldRemove(item)) toRemove.Add(item); });
foreach (var item in toRemove) list.Remove(item);
```

### 20.7 Избыточная синхронизация / слишком широкий lock

Держите критическую секцию максимально короткой; не делайте I/O или долгие вычисления внутри `lock`.

### 20.8 Deadlock от вложенных блокировок (lock ordering)

```csharp
// Поток A: lock(obj1) -> lock(obj2)
// Поток B: lock(obj2) -> lock(obj1)  <-- классический deadlock
```
**Решение:** всегда захватывайте несколько блокировок в едином глобальном порядке во всём приложении.

### 20.9 ConfigureAwait(false), а потом обращение к UI

Если поставили `ConfigureAwait(false)`, после `await` вы уже не гарантированно в UI-потоке — не трогайте элементы UI напрямую, используйте `Dispatcher.Invoke`/`SynchronizationContext.Post`.

### 20.10 Использование `Thread` напрямую без необходимости

`new Thread(...)` — редко нужно в прикладном коде. Пул потоков (`Task.Run`) эффективнее для коротких/средних задач. Настоящий `Thread` — только для долгоживущих выделенных потоков (например, STA-поток для COM).

---

## 21. Диагностика и отладка

- **Visual Studio Parallel Stacks / Tasks окна** — визуализация состояния задач и потоков при отладке.
- **`dotnet-trace` / `dotnet-counters`** — сбор трейсов и live-метрик (ThreadPool queue length, contention) в проде.
- **`EventCounters` / `ThreadPool.ThreadCount`, `PendingWorkItemCount`** — если пул потоков "голодает" (thread pool starvation), задачи копятся в очереди.
- **Contention:** `Monitor.LockContentionCount` — счётчик конкуренции за `lock`.
- **Проверка дедлоков:** снять dump процесса (`dotnet-dump`) в момент зависания и посмотреть стеки потоков — ищите потоки, ожидающие друг друга по кругу.
- **Логирование async-исключений:** подписка на `TaskScheduler.UnobservedTaskException` — ловит необработанные исключения в "забытых" задачах (не панацея, но полезно для диагностики).

```csharp
TaskScheduler.UnobservedTaskException += (sender, e) =>
{
    Log(e.Exception);
    e.SetObserved(); // предотвращает падение процесса на старых рантаймах
};
```

---

## 22. Выбор инструмента: таблица решений

| Задача | Инструмент |
|---|---|
| Запрос к БД/HTTP/файлу | `async/await`, `Task<T>` |
| Тяжёлые CPU-вычисления на весь список | `Parallel.For/ForEach`, PLINQ |
| CPU-вычисления + I/O внутри итерации | `Parallel.ForEachAsync` |
| Ограничить кол-во параллельных операций | `SemaphoreSlim` |
| Взаимоисключающий синхронный доступ к ресурсу | `lock` / `Monitor` |
| Взаимоисключающий доступ в async-методе | `SemaphoreSlim(1,1)` |
| Много читателей, редкие писатели | `ReaderWriterLockSlim` |
| Между процессами / машинами (один экземпляр) | `Mutex` (именованный) |
| Простой атомарный счётчик/флаг | `Interlocked`, `volatile` |
| Потокобезопасная коллекция общего назначения | `ConcurrentDictionary`, `ConcurrentQueue` |
| Producer/Consumer в async-коде | `Channel<T>` |
| Producer/Consumer в синхронном коде | `BlockingCollection<T>` |
| Многостадийный конвейер обработки | TPL Dataflow |
| Поток значений по мере готовности | `IAsyncEnumerable<T>` |
| Дождаться нескольких независимых операций | `Task.WhenAll` |
| Взять первый готовый результат / таймаут | `Task.WhenAny`, `.WaitAsync()` |
| Периодическая фоновая работа | `PeriodicTimer` |
| Однократная ленивая асинхронная инициализация | `Lazy<Task<T>>` |
| Отмена длительной операции | `CancellationToken` |

---

## 23. Чек-лист перед коммитом

- [ ] Все I/O-операции — `async`/`await`, ни одного `.Result`/`.Wait()` в асинхронном пути.
- [ ] Ни одного `async void`, кроме обработчиков событий UI.
- [ ] Публичные async-методы принимают `CancellationToken` и пробрасывают его дальше.
- [ ] Библиотечный код (не UI/ASP.NET Core контроллеры) использует `ConfigureAwait(false)`.
- [ ] Нет `Thread.Sleep` в асинхронном коде — только `Task.Delay`.
- [ ] Нет забытых `await` (нет предупреждений CS4014); осознанный fire-and-forget обёрнут в try/catch с логированием.
- [ ] Общее изменяемое состояние защищено (`lock`, `SemaphoreSlim`, `Interlocked` или потокобезопасная коллекция) — либо устранено вовсе.
- [ ] Внутри `lock` нет `await`, нет I/O, нет долгих вычислений.
- [ ] Порядок захвата нескольких блокировок одинаков во всём приложении (нет риска deadlock).
- [ ] `Parallel.For/PLINQ` используется для CPU-bound задач, а не для I/O.
- [ ] `MaxDegreeOfParallelism` задан там, где неконтролируемый параллелизм опасен.
- [ ] `ValueTask` не используется "по умолчанию" без реальной причины (профилирование/хот-пас).
- [ ] Исключения из `Task.WhenAll` для нескольких задач обрабатываются с учётом `AggregateException`/`InnerExceptions`, если это важно.
- [ ] `CancellationTokenSource`, `SemaphoreSlim`, `Mutex` и т.п. корректно `Dispose`/`using`.
- [ ] Для длительных фоновых процессов в ASP.NET Core используется `IHostedService`/`BackgroundService`, а не "голый" `Task.Run` в контроллере.

---

*Актуально для .NET 5–8+ (проверяйте документацию Microsoft Learn при использовании фич конкретной версии, например `Parallel.ForEachAsync` требует .NET 6+, `PeriodicTimer` — .NET 6+, `ConfigureAwaitOptions` — .NET 8+).*
