# Шпаргалка: асинхронное программирование в C# (.NET 5+)

Практический справочник для повседневного написания кода. Актуально для .NET 5–9.

---

## 1. Базовые принципы

### 1.1 Async — это про I/O, не про CPU
`async/await` не создаёт поток. Он освобождает поток на время ожидания I/O (сеть, диск, БД). Для CPU-bound работы используйте `Task.Run`, а не `async`.

```csharp
// I/O-bound — используем async нативно
public async Task<string> GetDataAsync() =>
    await httpClient.GetStringAsync(url);

// CPU-bound — выносим в пул потоков явно
public Task<int> ComputeAsync(int[] data) =>
    Task.Run(() => HeavyCompute(data));
```

### 1.2 Async всё до самого верха ("async all the way")
Не смешивайте синхронный и асинхронный код через `.Result`, `.Wait()`, `.GetAwaiter().GetResult()` — это частая причина дедлоков и истощения пула потоков.

```csharp
// Плохо
var result = GetDataAsync().Result;

// Хорошо
var result = await GetDataAsync();
```

### 1.3 Именование и сигнатуры
- Суффикс `Async` для всех асинхронных методов.
- Возвращайте `Task`/`Task<T>`/`ValueTask`/`ValueTask<T>` — никогда `void`, кроме обработчиков событий.
- Принимайте `CancellationToken` последним параметром, если метод может выполняться долго.

```csharp
Task<Order> GetOrderAsync(int id, CancellationToken ct = default);
```

---

## 2. async void — избегать

`async void` нельзя дождаться, исключения из него роняют процесс (через `SynchronizationContext` или `AppDomain.UnhandledException`), нет способа проверить завершение.

Единственное легитимное применение — обработчики событий UI (`void Button_Click`).

```csharp
// Плохо: необрабатываемое исключение уронит приложение
async void DoWork() => await RiskyAsync();

// Хорошо
async Task DoWorkAsync() => await RiskyAsync();
```

Если нужно вызвать async-метод из sync-контекста без ожидания — используйте `_ = FireAndForgetAsync();` с внутренним try/catch, либо `Task.Run` с логированием ошибок.

---

## 3. ConfigureAwait

### 3.1 Правило для библиотек
В коде библиотек/переиспользуемых компонентов всегда используйте `ConfigureAwait(false)`, чтобы не залипать на `SynchronizationContext` вызывающего кода (актуально для WinForms/WPF/классического ASP.NET).

```csharp
public async Task<Data> LoadAsync()
{
    var raw = await repository.GetAsync().ConfigureAwait(false);
    return Parse(raw);
}
```

### 3.2 В ASP.NET Core / worker-сервисах — не критично
В ASP.NET Core нет `SynchronizationContext`, поэтому `ConfigureAwait(false)` не обязателен, но не вреден. С .NET 6+ можно централизованно отключить контекст через `ConfigureAwaitOptions` (.NET 8+) точечно:

```csharp
await task.ConfigureAwait(ConfigureAwaitOptions.SuppressThrowing);
```

### 3.3 Итог
- Пишете библиотеку общего назначения → ставьте `.ConfigureAwait(false)` везде.
- Пишете ASP.NET Core приложение → можно опустить (но единообразие важнее).

---

## 4. Task vs ValueTask

| | `Task<T>` | `ValueTask<T>` |
|---|---|---|
| Аллокация | Всегда на куче | Может быть без аллокации (struct) |
| Повторное ожидание | Можно await много раз | **Нельзя** await дважды |
| Использование | Публичные API, кэшируемые результаты | Горячие пути с частым синхронным завершением |

```csharp
// Хороший кандидат на ValueTask: часто отдаёт кэш синхронно
public ValueTask<int> GetCachedAsync(string key)
{
    if (_cache.TryGetValue(key, out var value))
        return ValueTask.FromResult(value); // без аллокации Task

    return new ValueTask<int>(LoadAndCacheAsync(key));
}
```

Правила безопасности `ValueTask`:
- Не await'ить дважды.
- Не вызывать `.Result`/`.GetAwaiter().GetResult()` до завершения.
- Не сохранять для повторного использования — потребить один раз.
- Если сомневаетесь — используйте `Task<T>`, это безопаснее.

---

## 5. Отмена операций (CancellationToken)

### 5.1 Прокидывание токена
Всегда пробрасывайте `CancellationToken` вглубь вызовов, никогда не создавайте новый `CancellationToken.None` внутри метода, если снаружи он был передан.

```csharp
public async Task ProcessAsync(CancellationToken ct)
{
    await using var stream = await OpenAsync(ct);
    await stream.ReadAsync(buffer, ct);
}
```

### 5.2 Таймауты
```csharp
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(30));
await DoWorkAsync(cts.Token);

// .NET 6+: без ручного CTS
await task.WaitAsync(TimeSpan.FromSeconds(30));

// Комбинация токенов
using var linked = CancellationTokenSource.CreateLinkedTokenSource(userCt, timeoutCts.Token);
```

### 5.3 Обработка отмены
Ловите `OperationCanceledException` (базовый класс для `TaskCanceledException`), не глотайте её молча в бизнес-логике — часто нужно пробросить выше.

```csharp
try
{
    await DoWorkAsync(ct);
}
catch (OperationCanceledException) when (ct.IsCancellationRequested)
{
    logger.LogInformation("Операция отменена пользователем");
    throw;
}
```

### 5.4 Регистрация колбэка
```csharp
using var registration = ct.Register(() => connection.Abort());
```

---

## 6. Обработка исключений

### 6.1 AggregateException — только у Task.Wait/Result
`await` разворачивает `AggregateException` и бросает первое внутреннее исключение. `Task.Wait()`/`.Result` — нет, там нужно `.InnerException` или `.Flatten()`.

```csharp
try
{
    await Task.WhenAll(task1, task2);
}
catch (Exception ex)
{
    // Поймали только первое исключение! Остальные см. ниже.
}

// Чтобы собрать ВСЕ исключения из WhenAll:
var whenAllTask = Task.WhenAll(task1, task2);
try { await whenAllTask; }
catch
{
    var allErrors = whenAllTask.Exception?.InnerExceptions;
}
```

### 6.2 Необработанные исключения в fire-and-forget
```csharp
_ = Task.Run(async () =>
{
    try { await RiskyAsync(); }
    catch (Exception ex) { logger.LogError(ex, "Background task failed"); }
});
```

### 6.3 TaskScheduler.UnobservedTaskException
Подписка полезна как страховочная сетка для необработанных исключений в "забытых" тасках (не полагайтесь на неё как на основной механизм).

---

## 7. Комбинирование задач

### 7.1 Параллельное ожидание
```csharp
var usersTask = GetUsersAsync(ct);
var ordersTask = GetOrdersAsync(ct);

await Task.WhenAll(usersTask, ordersTask);

var users = await usersTask;   // уже завершены, без блокировки
var orders = await ordersTask;
```

### 7.2 Ограничение параллелизма — Parallel.ForEachAsync (.NET 6+)
Предпочтительнее ручных `SemaphoreSlim` + `Task.WhenAll` для типовых сценариев.

```csharp
await Parallel.ForEachAsync(
    items,
    new ParallelOptions { MaxDegreeOfParallelism = 8, CancellationToken = ct },
    async (item, token) => await ProcessAsync(item, token));
```

### 7.3 Ручное ограничение через SemaphoreSlim
```csharp
var semaphore = new SemaphoreSlim(8);
var tasks = items.Select(async item =>
{
    await semaphore.WaitAsync(ct);
    try { await ProcessAsync(item, ct); }
    finally { semaphore.Release(); }
});
await Task.WhenAll(tasks);
```

### 7.4 Первый завершённый — Task.WhenAny
```csharp
var completed = await Task.WhenAny(taskA, taskB);
if (completed == taskA) { /* ... */ }

// Осторожно: WhenAny не отменяет проигравшие задачи —
// используйте CancellationToken, чтобы не оставлять их висеть.
```

### 7.5 Таймаут для одиночной задачи
```csharp
// .NET 6+
try
{
    var result = await SlowOperationAsync(ct).WaitAsync(TimeSpan.FromSeconds(5), ct);
}
catch (TimeoutException)
{
    // не успели
}
```

---

## 8. Async streams и потоковая обработка

### 8.1 IAsyncEnumerable<T>
```csharp
public async IAsyncEnumerable<Order> GetOrdersAsync(
    [EnumeratorCancellation] CancellationToken ct = default)
{
    await foreach (var row in dbReader.ReadRowsAsync(ct))
    {
        yield return MapToOrder(row);
    }
}

// Потребление
await foreach (var order in GetOrdersAsync(ct))
{
    Process(order);
}
```

Важно: атрибут `[EnumeratorCancellation]` обязателен, чтобы токен, переданный в `WithCancellation`, реально пробрасывался внутрь итератора.

```csharp
await foreach (var order in GetOrdersAsync().WithCancellation(ct)) { }
```

### 8.2 System.Threading.Channels — производитель/потребитель
Замена `BlockingCollection` для async-сценариев.

```csharp
var channel = Channel.CreateBounded<int>(new BoundedChannelOptions(100)
{
    FullMode = BoundedChannelFullMode.Wait
});

// Producer
_ = Task.Run(async () =>
{
    foreach (var item in source)
        await channel.Writer.WriteAsync(item, ct);
    channel.Writer.Complete();
});

// Consumer
await foreach (var item in channel.Reader.ReadAllAsync(ct))
{
    Process(item);
}
```

---

## 9. Синхронизация и общее состояние

### 9.1 lock — не работает с await внутри
`lock` (Monitor) нельзя использовать вокруг `await` (с C# 13/.NET 9 `System.Threading.Lock` даёт понятную ошибку компиляции при попытке). Используйте `SemaphoreSlim`.

```csharp
private readonly SemaphoreSlim _lock = new(1, 1);

public async Task UpdateAsync()
{
    await _lock.WaitAsync();
    try { /* критическая секция с await */ }
    finally { _lock.Release(); }
}
```

### 9.2 Ленивая асинхронная инициализация
```csharp
private readonly Lazy<Task<Config>> _config =
    new(() => LoadConfigAsync(), LazyThreadSafetyMode.ExecutionAndPublication);

public Task<Config> GetConfigAsync() => _config.Value;
```

### 9.3 TaskCompletionSource — мост между событиями и Task
```csharp
var tcs = new TaskCompletionSource<int>(TaskCreationOptions.RunContinuationsAsynchronously);

client.OnResponse += (s, e) => tcs.TrySetResult(e.Value);
client.OnError += (s, e) => tcs.TrySetException(e.Exception);

var result = await tcs.Task;
```
`RunContinuationsAsynchronously` обязателен — без него continuation может выполниться синхронно в потоке, вызвавшем `SetResult`, что ведёт к дедлокам/неожиданному ре-энтерансу.

---

## 10. Async-ресурсы и IAsyncDisposable

```csharp
public class Connection : IAsyncDisposable
{
    public async ValueTask DisposeAsync()
    {
        await FlushAsync();
        _socket.Dispose();
    }
}

await using var conn = new Connection();
```

`await using` вызывает `DisposeAsync` асинхронно даже при исключении внутри блока.

---

## 11. Частые ловушки и антипаттерны

| Антипаттерн | Проблема | Решение |
|---|---|---|
| `.Result` / `.Wait()` | Дедлок при наличии SynchronizationContext, блокирует поток | `await` |
| `async void` | Исключения роняют процесс | `async Task` |
| `Task.Run` внутри ASP.NET-контроллера для I/O | Тратит поток впустую, I/O и так асинхронный | Просто `await` |
| Игнорирование `CancellationToken` | Утечка ресурсов, зависшие запросы | Пробрасывать токен everywhere |
| `catch (Exception)` без `when` вокруг отменяемого кода | Отмена выглядит как ошибка | Отдельно ловить `OperationCanceledException` |
| Двойной `await` `ValueTask` | UB, может упасть | Использовать `Task` или потребить один раз |
| `async` метод без `await` внутри | Warning CS1998, синхронное выполнение с оверхедом Task | Либо добавить await, либо вернуть `Task.FromResult` явно |
| Захват `foreach`-переменной в замыкании до C# 5 | Устарело с C# 5+, но проверяйте legacy-код | Актуально только для .NET Framework <4.5 |
| Смешение `Task<T>` и `async Task<T>` без надобности | Лишний state machine | Если тело — один `return await`/один вызов, можно вернуть Task напрямую (см. 11.1) |

### 11.1 Когда можно не писать async/await
```csharp
// Без лишней state machine — просто пробрасываем Task
public Task<User> GetUserAsync(int id) => repository.GetUserAsync(id);

// Но если нужен try/finally, using, ConfigureAwait — тогда async/await обязателен,
// иначе стектрейс исключения будет менее информативным и using не сработает как ожидается.
```

---

## 12. ASP.NET Core — специфика

### 12.1 Не блокировать поток запроса
```csharp
// Плохо — блокирует поток из пула, снижает throughput
public IActionResult Get() => Ok(_service.GetDataAsync().Result);

// Хорошо
public async Task<IActionResult> Get() => Ok(await _service.GetDataAsync());
```

### 12.2 Background-задачи в веб-приложении
Используйте `IHostedService`/`BackgroundService`, а не `Task.Run` "в никуда" из контроллера — иначе таск может быть прерван при завершении обработки запроса.

```csharp
public class QueueWorker : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await ProcessQueueAsync(stoppingToken);
        }
    }
}
```

Если всё же нужно выполнить работу после ответа — используйте `IHostApplicationLifetime` + `IBackgroundTaskQueue`, а не голый `Task.Run`.

### 12.3 HttpClient — переиспользование
Используйте `IHttpClientFactory`, не создавайте `new HttpClient()` на каждый запрос (исчерпание сокетов).

```csharp
services.AddHttpClient<IMyApiClient, MyApiClient>()
    .AddPolicyHandler(Polly.Policy...); // ретраи/таймауты через Polly
```

---

## 13. Таймеры и периодические задачи

### 13.1 PeriodicTimer (.NET 6+) — предпочтительнее System.Timers.Timer
```csharp
using var timer = new PeriodicTimer(TimeSpan.FromSeconds(10));
while (await timer.WaitForNextTickAsync(ct))
{
    await DoPeriodicWorkAsync(ct);
}
```
Не накапливает "просроченные" тики, корректно работает с `await` внутри, легко останавливается через `CancellationToken`.

---

## 14. Тестирование асинхронного кода

```csharp
[Fact]
public async Task GetOrderAsync_ReturnsOrder()
{
    var result = await _service.GetOrderAsync(1);
    Assert.NotNull(result);
}

// Проверка исключения
[Fact]
public async Task Throws_OnInvalidId()
{
    await Assert.ThrowsAsync<NotFoundException>(() => _service.GetOrderAsync(-1));
}

// Таймаут в тесте, чтобы не зависал CI
[Fact(Timeout = 5000)]
public async Task DoesNotHang() => await _service.RunAsync();
```

Никогда не используйте `.Result`/`.Wait()` в тестах — тестовый метод сам должен быть `async Task`.

---

## 15. Производительность: чек-лист

1. Не создавайте лишние `async` state machine на "горячем пути", если можно вернуть Task напрямую (п. 11.1).
2. Используйте `ValueTask` там, где синхронное завершение — частый случай (кэши, буферы).
3. Не аллоцируйте замыкания в горячих циклах `async` лямбд без необходимости.
4. Для конкатенации асинхронных операций над коллекциями предпочитайте `Task.WhenAll` вместо последовательных `await` в цикле, если операции независимы.
5. Ограничивайте параллелизм (`Parallel.ForEachAsync`, `SemaphoreSlim`) — неограниченный `Task.WhenAll` на тысячах элементов может исчерпать сокеты/подключения к БД.
6. Профилируйте через `dotnet-trace`/`dotnet-counters` — метрика `ThreadPool Queue Length` покажет истощение пула из-за sync-over-async.
7. С .NET 6+ ThreadPool быстрее наращивает потоки (`min threads` менее критичен), но избегание блокировок остаётся правилом №1.

---

## 16. Быстрый справочник по API (.NET 5–9)

| API | Доступно с | Назначение |
|---|---|---|
| `Task.WhenAll` / `WhenAny` | всегда | Комбинирование задач |
| `Task.Run` | всегда | Смещение CPU-bound работы в пул потоков |
| `IAsyncEnumerable<T>` | C# 8 / .NET Core 3 | Асинхронные потоки данных |
| `System.Threading.Channels` | .NET Core 3+ | Producer/consumer без блокировок |
| `Task.WaitAsync(timeout, ct)` | .NET 6 | Таймаут/отмена для готовой Task |
| `PeriodicTimer` | .NET 6 | Периодические async-операции |
| `Parallel.ForEachAsync` | .NET 6 | Ограниченный параллелизм по коллекции |
| `ConfigureAwaitOptions` | .NET 8 | Точечная настройка await (suppress throwing и др.) |
| `System.Threading.Lock` | C# 13 / .NET 9 | Более быстрый и безопасный аналог `lock`, с защитой от `await` внутри |
| `TaskCompletionSource<T>` | всегда (типизированный с .NET 4.5+) | Мост callback → Task |

---

## 17. Мини-чеклист перед коммитом async-кода

- [ ] Нет `.Result`, `.Wait()`, `.GetAwaiter().GetResult()` в асинхронном пути.
- [ ] Нет `async void`, кроме UI event handlers.
- [ ] `CancellationToken` принимается и пробрасывается везде, где это уместно.
- [ ] Библиотечный код использует `ConfigureAwait(false)`.
- [ ] `ValueTask` не await'ится дважды и не сохраняется для повторного использования.
- [ ] Исключения из fire-and-forget тасков логируются, а не теряются.
- [ ] `IAsyncDisposable`-ресурсы освобождаются через `await using`.
- [ ] Нет неограниченного параллелизма на коллекциях (используется семафор/`Parallel.ForEachAsync`).
- [ ] Тесты для async-методов сами `async Task`, без блокирующих вызовов.
