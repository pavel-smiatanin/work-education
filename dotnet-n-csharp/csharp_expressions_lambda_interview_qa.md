# C# Expressions & Lambda Expressions — вопросы для собеседований (Intermediate → Advanced → Guru)

> 120+ вопросов, сгруппированных по темам: от базового синтаксиса лямбд до построения expression trees вручную,
> вариантности делегатов и подводных камней производительности. У каждого вопроса указан уровень сложности,
> развёрнутый ответ с примерами кода, ссылки на официальную документацию и (где это помогает) диаграмма/схема.

**Легенда уровней:** 🟢 Intermediate · 🟡 Advanced · 🔴 Guru

---

## Оглавление

1. [Основы лямбда-выражений](#1-основы-лямбда-выражений)
2. [Делегаты и совместимость типов с лямбдами](#2-делегаты-и-совместимость-типов-с-лямбдами)
3. [Замыкания и захват переменных](#3-замыкания-и-захват-переменных)
4. [Expression-bodied members](#4-expression-bodied-members)
5. [LINQ и лямбды](#5-linq-и-лямбды)
6. [Отложенное выполнение: IEnumerable vs IQueryable](#6-отложенное-выполнение-ienumerable-vs-iqueryable)
7. [Expression Trees](#7-expression-trees)
8. [Func / Action / Predicate и пользовательские делегаты](#8-func--action--predicate-и-пользовательские-делегаты)
9. [Локальные функции vs лямбды](#9-локальные-функции-vs-лямбды)
10. [Асинхронные лямбды и Task](#10-асинхронные-лямбды-и-task)
11. [Pattern matching и switch expressions](#11-pattern-matching-и-switch-expressions)
12. [Другие выражения C#](#12-другие-выражения-c)
13. [Производительность и best practices](#13-производительность-и-best-practices)
14. [Advanced/Guru: построение expression trees, вариантность, static-лямбды и другое](#14-advancedguru-темы)

---

## 1. Основы лямбда-выражений

### Вопрос 1.1 🟢 Что такое лямбда-выражение в C# и зачем оно нужно?

**Ответ.** Лямбда-выражение — это компактный синтаксис для создания анонимной функции: кода, который можно передать как значение (в переменную, параметр метода, поле), не объявляя для него отдельный именованный метод. Синтаксис строится вокруг оператора `=>` ("goes to"), который отделяет список входных параметров от тела:

```csharp
x => x * 2                     // expression lambda: тело — выражение
(x, y) => { return x + y; }    // statement lambda: тело — блок операторов
```

Главная цель — избавиться от бойлерплейта: вместо отдельного метода с именем достаточно инлайн-функции там, где она используется один раз (фильтрация коллекции, обработчик события, задача для `Task.Run`). Лямбда конвертируется либо в делегат (`Func<>`, `Action<>`, свой делегат), либо в expression tree (`Expression<TDelegate>`), в зависимости от типа, к которому её приводят.

**Ресурсы.**
- [Lambda expressions and anonymous functions](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/lambda-expressions)
- [Lambda expressions, delegates, and events](https://learn.microsoft.com/dotnet/csharp/fundamentals/types/delegates-lambdas)

---

### Вопрос 1.2 🟢 Чем expression lambda отличается от statement lambda?

**Ответ.** *Expression lambda* — тело состоит из одного выражения без фигурных скобок и `return`; результатом лямбды становится значение этого выражения: `(input) => expression`. *Statement lambda* — тело обёрнуто в фигурные скобки и может содержать произвольное число операторов (`{ statement1; statement2; ... }`), допускается использовать `return` явно.

Ключевое отличие с практическими последствиями: **компилятор может построить expression tree только из expression lambda**. Statement lambda конвертируется исключительно в делегат — её нельзя присвоить переменной типа `Expression<TDelegate>`.

```csharp
Expression<Func<int,int>> ok = x => x * x;               // компилируется
Expression<Func<int,int>> bad = x => { return x * x; };  // ошибка компиляции
```

**Ресурсы.**
- [Expression lambdas / Statement lambdas](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/lambda-expressions#expression-lambdas)

---

### Вопрос 1.3 🟢 Как записываются параметры лямбда-выражений (0, 1, несколько, с типами)?

**Ответ.**
- Без параметров: `() => Console.WriteLine("hi")`.
- Один параметр — скобки можно опустить: `x => x * x`.
- Несколько параметров — скобки обязательны, через запятую: `(x, y) => x + y`.
- Явные типы параметров (все или ни одного — либо всё implicit, либо всё explicit): `(int x, string s) => s.Length > x`.
- Значения по умолчанию (с C# 12) и `params`-массивы/коллекции (с C# 12/13) при явной типизации:

```csharp
var incrementBy = (int source, int increment = 1) => source + increment;
var sum = (params IEnumerable<int> values) => values.Sum();
```

Важный нюанс: лямбды со значениями по умолчанию или `params` не имеют "естественного типа" `Func<>`/`Action<>` — их нужно присваивать через `var` (компилятор синтезирует подходящий тип делегата) либо через собственный `delegate`.

**Ресурсы.**
- [Input parameters of a lambda expression](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/lambda-expressions#input-parameters-of-a-lambda-expression)

---

### Вопрос 1.4 🟢 Как работает вывод типов (type inference) в лямбдах, и когда типы нужно указывать явно?

**Ответ.** Компилятор выводит типы параметров лямбды из контекста — обычно из сигнатуры делегата, к которому лямбда приводится:

```csharp
Func<int,int> square = x => x * x; // x выведен как int из Func<int,int>
```

Если контекста недостаточно (например, `var parse = s => int.Parse(s);` без указания целевого типа делегата), компилятор не может вывести тип `s`, и возникает ошибка компиляции. В этом случае нужно указать тип делегата явно: `Func<string,int> parse = s => int.Parse(s);`.

Аналогично иногда неочевиден **тип возврата**: `var choose = (bool b) => b ? 1 : "two";` не компилируется, так как `1` и `"two"` не имеют общего типа. Можно указать явный тип возврата перед списком параметров (с C# 10): `var choose = object (bool b) => b ? 1 : "two";`.

**Ресурсы.**
- [Natural type of a lambda expression](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/lambda-expressions#natural-type-of-a-lambda-expression)
- [Explicit return type](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/lambda-expressions#explicit-return-type)

---

### Вопрос 1.5 🟡 Что такое "естественный тип" (natural type) лямбда-выражения и когда его нет?

**Ответ.** Начиная с C# 10, лямбда-выражение само по себе может иметь тип (в отличие от прежних версий, где лямбда была типизируема только в контексте присваивания). Если типы параметров и возврата выводимы без внешнего контекста, компилятор присваивает лямбде естественный тип `Func<...>`/`Action<...>` или (для var в контексте `Expression`) `Expression<Func<...>>`. Это позволяет писать:

```csharp
var read = (string s) => int.Parse(s);   // var + explicit-typed lambda → Func<string,int>
```

Естественного типа **нет**, если:
- параметры не типизированы явно и нет целевого контекста (`var f = x => x + 1;` — ошибка);
- тело — условное выражение с несовместимыми ветками без общего типа;
- лямбда использует значения по умолчанию параметров или `params` (тогда `var` создаёт анонимный delegate-тип, а не `Func<>`/`Action<>`).

**Ресурсы.**
- [Natural type of a lambda expression](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/lambda-expressions#natural-type-of-a-lambda-expression)

---

### Вопрос 1.6 🟢 Можно ли использовать discard (`_`) в параметрах лямбды и зачем?

**Ответ.** Да. Discard-параметр `_` сигнализирует, что значение не используется в теле. Особенно полезно для обработчиков событий или делегатов с несколькими параметрами, из которых нужен только один:

```csharp
Action<int,int,string> statusUpdate = (_, _, message) => Console.WriteLine(message);
button.Click += (_, _) => DoSomething();
```

Особенность: если единственный параметр лямбды назван `_`, компилятор из соображений обратной совместимости трактует его как обычное имя параметра (а не discard) внутри данной лямбды. Discard как признак "два и более `_`" появился, чтобы избежать конфликта имён (CS0136), — иначе нельзя было бы объявить два параметра с одинаковым именем `_`.

**Ресурсы.**
- [Discards](https://learn.microsoft.com/dotnet/csharp/fundamentals/functional/discards)
- [Input parameters — discards](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/lambda-expressions#input-parameters-of-a-lambda-expression)

---

### Вопрос 1.7 🟡 Можно ли навесить атрибуты на лямбда-выражение, его параметры или возвращаемое значение?

**Ответ.** Да, начиная с C# 10:

```csharp
Func<string?, int?> parse = [ProvidesNullCheck] (s) => (s is not null) ? int.Parse(s) : null;
var concat = ([DisallowNull] string a, [DisallowNull] string b) => a + b;
var inc = [return: NotNullIfNotNull(nameof(s))] (int? s) => s.HasValue ? s + 1 : null;
```

При этом параметры обязательно нужно заключить в скобки, даже если параметр один. **Важная ловушка на собеседовании**: атрибуты на лямбде не влияют на её вызов — вызов идёт через `Invoke` делегата, который атрибуты не проверяет и не применяет. Поэтому, например, `[Conditional]` на лямбде не работает — атрибуты полезны только для статического анализа и рефлексии.

**Ресурсы.**
- [Attributes on lambda expressions](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/lambda-expressions#attributes)

---

### Вопрос 1.8 🟢 В чём разница между лямбда-выражением и анонимным методом (`delegate { }`)?

**Ответ.** Анонимные методы — более старый (C# 2.0) способ создать инлайн-делегат:

```csharp
Func<int,int> square = delegate (int x) { return x * x; };
```

Отличия от лямбд:
- Лямбды короче и поддерживают вывод типов параметров; анонимные методы требуют явных типов (кроме случая без параметров вообще).
- Анонимный метод можно объявить **без списка параметров вовсе** (`delegate { ... }`), и тогда он совместим с делегатом любой сигнатуры — параметры просто игнорируются. У лямбд так нельзя.
- Анонимные методы **нельзя** конвертировать в expression trees — только в делегаты.
- Внутри современного кода анонимные методы практически не используются — их вытеснили лямбды, но они остаются в языке для обратной совместимости.

**Ресурсы.**
- [Anonymous methods (delegate operator)](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/delegate-operator)

---

### Вопрос 1.9 🟢 Как лямбда используется как аргумент метода, ожидающего делегат?

**Ответ.** Когда метод объявляет параметр типа `Func<>`/`Action<>`/произвольного делегата, вызывающий код передаёт лямбду, чья сигнатура совпадает по числу, типам параметров и типу возврата:

```csharp
static IEnumerable<int> Filter(IEnumerable<int> source, Func<int,bool> predicate)
{
    foreach (var item in source)
        if (predicate(item))
            yield return item;
}

var evens = Filter(numbers, value => value % 2 == 0);
```

Компилятор проверяет соответствие сигнатур на этапе компиляции — если лямбда возвращает не тот тип или принимает не то число параметров, код не скомпилируется. Этот паттерн лежит в основе почти всего LINQ и множества API (`Task.Run`, обработчики событий, `List<T>.Sort(Comparison<T>)`, `Array.Find(Predicate<T>)`).

**Ресурсы.**
- [Pass a lambda expression to a method](https://learn.microsoft.com/dotnet/csharp/fundamentals/types/delegates-lambdas#pass-a-lambda-expression-to-a-method)

---

### Вопрос 1.10 🟡 Как выглядит лямбда, представленная как `Expression<Func<>>`, при выводе через `ToString()`/`Console.WriteLine`?

**Ответ.** Когда лямбда компилируется в expression tree, а не в делегат, объект `Expression<TDelegate>` можно вывести на печать — `ToString()` отрендерит человекочитаемое представление дерева:

```csharp
Expression<Func<int,int>> e = x => x * x;
Console.WriteLine(e);
// Вывод: x => (x * x)
```

Это полезно для отладки: видно, как компилятор расставил скобки согласно приоритету операторов, и какие узлы дерева были построены. При этом `e` — это не исполняемый код, а *данные*, описывающие код (см. раздел про Expression Trees); чтобы выполнить его, нужно вызвать `e.Compile()`.

**Ресурсы.**
- [Lambda expressions — printing an expression tree](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/lambda-expressions)

---

## 2. Делегаты и совместимость типов с лямбдами

### Вопрос 2.1 🟢 Что такое делегат и как он связан с лямбдами?

**Ответ.** Делегат — тип, представляющий сигнатуру метода (список параметров + тип возврата). Переменная типа делегата может хранить ссылку на любой метод (именованный, лямбду, анонимный метод), подходящий под эту сигнатуру:

```csharp
delegate int Transform(int value);

Transform doubler = x => x * 2;       // лямбда
Transform squarer = Square;           // именованный метод (method group)
static int Square(int v) => v * v;
```

Чтобы использовать лямбду, компилятору нужно знать типы параметров и тип возврата — это описание и есть тип делегата. Лямбда компилируется либо в статический метод (если ничего не захватывает), либо в метод сгенерированного класса-замыкания, а делегат оборачивает ссылку на этот метод (и, при необходимости, на экземпляр-замыкание).

**Ресурсы.**
- [Delegates support lambda expressions](https://learn.microsoft.com/dotnet/csharp/fundamentals/types/delegates-lambdas#delegates-support-lambda-expressions)

---

### Вопрос 2.2 🟢 Чем отличаются `Func<T>`, `Action<T>` и `Predicate<T>`?

**Ответ.**
- `Func<T1,...,TResult>` — делегат, **возвращающий** значение типа `TResult`; последний параметр типа всегда результат. `Func<int,bool> isEven = x => x % 2 == 0;`
- `Action<T1,...>` — делегат, **не возвращающий** значения (`void`). `Action<string> log = s => Console.WriteLine(s);`
- `Predicate<T>` — специализированный делегат `bool Predicate<T>(T obj)`, семантически равен `Func<T,bool>`, но не взаимозаменяем с ним напрямую (нужно явное приведение/новая лямбда), исторически использовался в `List<T>.Find`, `Array.FindAll` до появления LINQ.

На собеседовании часто спрашивают: "можно ли присвоить `Predicate<int>` переменной `Func<int,bool>`?" — нет напрямую, так как это разные типы делегатов, даже с одинаковой сигнатурой. Нужно обернуть в новую лямбду: `Func<int,bool> f = x => predicate(x);`.

**Ресурсы.**
- [Func<T,TResult>](https://learn.microsoft.com/dotnet/api/system.func-2)
- [Predicate<T>](https://learn.microsoft.com/dotnet/api/system.predicate-1)

---

### Вопрос 2.3 🟡 Как объявить и использовать собственный тип делегата вместо `Func`/`Action`?

**Ответ.** Пользовательский делегат оправдан, когда сигнатура часто повторяется и важна семантика имени, либо нужны `ref`/`out`/`in` параметры (которые `Func`/`Action` не поддерживают), либо нужны значения по умолчанию/`params`:

```csharp
public delegate bool TryParseHandler<T>(string input, out T result);
public delegate int SumDelegate(params int[] values);
```

`Func`/`Action` покрывают 95% случаев, но собственный делегат делает код более выразительным (`ValidationRule`, `EventHandler<T>`) и единственно возможен там, где нужны `ref`/`out`/`in` параметры или non-Func-совместимые сигнатуры.

**Ресурсы.**
- [Delegates (C# Programming Guide)](https://learn.microsoft.com/dotnet/csharp/programming-guide/delegates/)

---

### Вопрос 2.4 🟡 Что такое multicast delegate, и как ведут себя `Func`-делегаты с несколькими подписчиками?

**Ответ.** Делегат в .NET — multicast: оператор `+=`/`Combine` добавляет ещё один метод в список вызова (`GetInvocationList()`). Для `Action` это стандартный паттерн событий — вызываются все подписчики по порядку. Для `Func<T,TResult>` с несколькими подписчиками **результат вызова — результат только последнего** метода в списке; промежуточные результаты отбрасываются (хотя все методы выполняются):

```csharp
Func<int,int> f = x => x + 1;
f += x => x * 10;
Console.WriteLine(f(5));  // 50 — вернулся результат ВТОРОГО делегата, первый выполнился, но его результат пропал
```

Это частая ловушка: если нужно собрать несколько результатов, используйте `GetInvocationList()` и вызывайте каждый делегат вручную, либо `event` с обработкой через `Delegate.GetInvocationList()`.

**Ресурсы.**
- [Delegate.Combine](https://learn.microsoft.com/dotnet/api/system.delegate.combine)

---

### Вопрос 2.5 🟢 Как метод (method group) конвертируется в делегат без явной лямбды?

**Ответ.** Если сигнатура именованного метода совпадает с сигнатурой делегата, метод можно присвоить напрямую — это называется **method group conversion**:

```csharp
static bool IsEven(int x) => x % 2 == 0;
Func<int,bool> pred = IsEven;             // без лямбды
var evens = numbers.Where(IsEven);        // передача method group в LINQ
```

Компилятор создаёт делегат, указывающий на этот метод. Для статических методов это не требует замыкания и не создаёт аллокаций на каждый вызов (в отличие от многократного создания лямбды в цикле — см. вопрос про производительность).

**Ресурсы.**
- [Method group conversions — C# language specification](https://learn.microsoft.com/dotnet/csharp/language-reference/language-specification/conversions)

---

### Вопрос 2.6 🟡 Что происходит, если делегат содержит null, и как безопасно вызывать событие?

**Ответ.** Вызов `null`-делегата (`myDelegate()`) бросает `NullReferenceException`. Классический безопасный паттерн — скопировать в локальную переменную и проверить на `null` перед вызовом (защита от гонки, когда другой поток отписался между проверкой и вызовом):

```csharp
var handler = SomeEvent;
handler?.Invoke(this, EventArgs.Empty);
```

Начиная с C# 6, `?.Invoke(...)` компилируется так, что operand читается один раз — это устраняет классическую гонку данных при прямой проверке `if (SomeEvent != null) SomeEvent(...)`, где между проверкой и вызовом другой поток мог обнулить делегат.

**Ресурсы.**
- [Member access operators — thread-safe delegate invocation](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/member-access-operators#null-conditional-operators--and-)

---

### Вопрос 2.7 🟡 Может ли лямбда быть присвоена переменной типа `object` или `Delegate`? Что тогда с ней можно (и нельзя) делать?

**Ответ.** Да, лямбда, будучи типизированной по контексту, может быть присвоена `Delegate` (базовый класс всех делегатов) или `object` — компилятор сначала должен вывести конкретный тип делегата (обычно `Func<>`/`Action<>`), а затем неявно привести к `Delegate`/`object`. Но **нельзя** написать `var x = () => 42;` без контекста, потому что компилятору не из чего вывести тип делегата.

Также лямбду/анонимный метод/method group **нельзя** использовать как левый операнд `is`/`as` (ошибка CS0837) — у них нет типа времени выполнения, который можно протестировать таким способом; нужно сначала присвоить переменной конкретного делегатного типа.

**Ресурсы.**
- [Errors and warnings — lambda expressions](https://learn.microsoft.com/dotnet/csharp/language-reference/compiler-messages/lambda-expression-errors#syntax-limitations-in-lambda-expressions)

---

### Вопрос 2.8 🟢 Как использовать лямбду в качестве `IComparer<T>`/`Comparison<T>` для сортировки?

**Ответ.**

```csharp
var people = new List<Person>();
people.Sort((a, b) => a.Age.CompareTo(b.Age));       // Comparison<Person>
people.OrderBy(p => p.Age).ThenBy(p => p.Name);      // LINQ, key selector
```

`List<T>.Sort` принимает `Comparison<T>` напрямую как делегат — лямбда идеально сюда подходит. Если же API требует именно интерфейс `IComparer<T>` (например, `SortedSet<T>`), нужен адаптер — `Comparer<T>.Create(comparison)`, который оборачивает `Comparison<T>` в `IComparer<T>`.

**Ресурсы.**
- [Comparer<T>.Create](https://learn.microsoft.com/dotnet/api/system.collections.generic.comparer-1.create)

---

### Вопрос 2.9 🟢 Как работает передача лямбды в `Task.Run`, `Parallel.Invoke`, `Timer`?

**Ответ.** Эти API принимают `Action`/`Func<T>` и лямбда естественно подставляется:

```csharp
await Task.Run(() => HeavyComputation());
Parallel.Invoke(() => DoWork1(), () => DoWork2());
```

Практический нюанс: замыкание, захватывающее переменные из внешнего метода, продлевает их жизнь до завершения задачи (объект-замыкание живёт, пока жив делегат). Если внутри цикла создаётся много задач с замыканиями на большие объекты, это может влиять на память — стоит по возможности передавать данные через параметры, а не захватывать напрямую.

**Ресурсы.**
- [Task-based asynchronous programming](https://learn.microsoft.com/dotnet/standard/parallel-programming/task-based-asynchronous-programming)

---

### Вопрос 2.10 🟡 Может ли лямбда быть рекурсивной? Как реализовать рекурсию через лямбду?

**Ответ.** Напрямую лямбда не может ссылаться сама на себя в момент объявления, так как переменная ещё не инициализирована. Но можно, объявив переменную заранее и присвоив рекурсивное тело после:

```csharp
Func<int,int> factorial = null!;
factorial = n => n <= 1 ? 1 : n * factorial(n - 1);
```

Это работает благодаря замыканию: лямбда захватывает саму переменную `factorial` (а не её значение на момент объявления), и к моменту вызова переменная уже содержит ссылку на саму лямбду. Более идиоматичный способ для рекурсии — **локальная функция** (см. раздел 9), которая поддерживает самоссылку без этого трюка и без обязательной аллокации делегата.

**Ресурсы.**
- [Local functions vs lambda expressions](https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/local-functions#local-functions-vs-lambda-expressions)

---

## 3. Замыкания и захват переменных

### Вопрос 3.1 🟢 Что такое замыкание (closure) в C#?

**Ответ.** Замыкание — это лямбда (или анонимный метод) вместе с внешними переменными, которые она "захватила" из окружающего кода. Захваченная переменная — это переменная, объявленная вне тела лямбды, но используемая внутри неё:

```csharp
int factor = 2;
Func<int,int> multiply = x => x * factor;   // factor захвачена
factor = 10;
Console.WriteLine(multiply(5));             // 50, а не 10 — используется ТЕКУЩЕЕ значение factor
```

Технически компилятор генерирует скрытый класс ("closure class"), в котором захваченные переменные становятся полями; сама лямбда становится методом этого класса. Все ссылки на захваченную переменную (и внутри лямбды, и в окружающем коде после её объявления) перенаправляются на поле этого объекта — поэтому изменение переменной снаружи видно внутри лямбды и наоборот.

**Диаграмма.**
```
До компиляции:                       После компиляции (упрощённо):

int factor = 2;                      class <>c__DisplayClass0
Func<int,int> multiply =             {
    x => x * factor;                     public int factor;
                                          public int <M>b__0(int x) => x * factor;
                                      }
                                      var d = new <>c__DisplayClass0();
                                      d.factor = 2;
                                      Func<int,int> multiply = d.<M>b__0;
```

**Ресурсы.**
- [Capture of outer variables and variable scope in lambda expressions](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/lambda-expressions#capture-of-outer-variables-and-variable-scope-in-lambda-expressions)
- [C# language specification — 12.22.6.2 Captured outer variables](https://learn.microsoft.com/dotnet/csharp/language-reference/language-specification/expressions#1222-anonymous-function-expressions)

---

### Вопрос 3.2 🟡 Продлевает ли захват переменной её время жизни? Приведите пример.

**Ответ.** Да. Захваченная переменная не уничтожается при выходе из метода — её время жизни продлевается минимум до тех пор, пока делегат (или expression tree), созданный из лямбды, не станет доступен для сборки мусора:

```csharp
static Func<int> Counter()
{
    int count = 0;
    return () => ++count;
}

var c = Counter();
Console.WriteLine(c());  // 1
Console.WriteLine(c());  // 2
Console.WriteLine(c());  // 3
```

Хотя `Counter()` завершился, `count` "живёт" внутри объекта-замыкания, на который ссылается возвращённый делегат `c`. Это классический вопрос на собеседовании — многие ожидают ошибку компиляции или что `count` каждый раз будет 0, но C# гарантирует корректное продление жизни переменной.

**Ресурсы.**
- [C# language specification — captured outer variables example](https://learn.microsoft.com/dotnet/csharp/language-reference/language-specification/expressions#1222-anonymous-function-expressions)

---

### Вопрос 3.3 🔴 Классическая ловушка: захват переменной цикла `for` vs `foreach` в разных версиях C#. Как она проявляется?

**Ответ.** До C# 5 переменная итерации `foreach` была **одной и той же** переменной на все итерации (как и переменная в классическом `for`), поэтому все лямбды, созданные в цикле, захватывали одну и ту же ячейку памяти, и после завершения цикла все они "видели" последнее значение:

```csharp
// Поведение ДО C# 5 (foreach) — все делегаты печатали 3, 3, 3
var actions = new List<Action>();
foreach (var i in new[] {1, 2, 3})
    actions.Add(() => Console.Write(i));
foreach (var a in actions) a();   // C# 5+: 1 2 3   |   до C# 5: 3 3 3
```

Начиная с **C# 5.0**, компилятор создаёт **новую переменную итерации на каждую итерацию `foreach`** (для `for` поведение не менялось — переменная там всё ещё одна на весь цикл, так как она объявлена вне тела цикла синтаксически). Поэтому для классического `for` проблема существует и сегодня:

```csharp
var actions = new List<Action>();
for (int i = 0; i < 3; i++)
    actions.Add(() => Console.Write(i));
foreach (var a in actions) a();   // всегда выведет 3 3 3 — i одна и та же для всех замыканий
```

**Как избежать:** скопировать переменную в новую локальную внутри тела цикла:

```csharp
for (int i = 0; i < 3; i++)
{
    int captured = i;
    actions.Add(() => Console.Write(captured));
}
```

**Диаграмма (два сценария захвата).**
```
foreach (C# 5+): каждая итерация — новая переменная       for: одна переменная на весь цикл
┌──────────┐ ┌──────────┐ ┌──────────┐                    ┌────────────────────────────┐
│ i = 1    │ │ i = 2    │ │ i = 3    │                    │ i (общая ячейка, меняется)  │
│ lambda1 ─┼─┘lambda2 ──┼─┘lambda3 ──┘                    │  ↑        ↑        ↑        │
└──────────┘ └──────────┘ └──────────┘                    │lambda1 lambda2  lambda3     │
   печатают: 1  2  3                                       └────────────────────────────┘
                                                              все три видят финальное i=3
```

**Ресурсы.**
- [C# version history — фикс foreach в C# 5](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-version-history)
- [Capture of outer variables and variable scope](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/lambda-expressions#capture-of-outer-variables-and-variable-scope-in-lambda-expressions)

---

### Вопрос 3.4 🟡 Можно ли захватить `ref`, `out`, `in` параметр или `ref`-локальную переменную в лямбде?

**Ответ.** Нет. Лямбда не может напрямую захватывать `ref`/`out`/`in` параметры внешнего метода и не может использовать `ref`-локальные переменные внутри себя (ошибки компиляции CS1628, CS8175). Причина — семантика ссылки на стек не может быть безопасно перенесена в объект-замыкание, живущий в куче и потенциально дольше самого стекового фрейма.

Решение — скопировать значение в обычную локальную переменную перед лямбдой:

```csharp
void M(ref int x)
{
    int copy = x;               // копируем значение
    Action a = () => Console.WriteLine(copy);
}
```

Либо использовать локальную функцию, которая может обращаться к `ref`/`out` параметрам напрямую (не будучи преобразованной в делегат), так как компилируется иначе.

**Ресурсы.**
- [Errors and warnings — CS1628, CS8175](https://learn.microsoft.com/dotnet/csharp/language-reference/compiler-messages/lambda-expression-errors#syntax-limitations-in-lambda-expressions)

---

### Вопрос 3.5 🟡 Как лямбда захватывает `this` в методе экземпляра, и в чём подвох для структур (`struct`)?

**Ответ.** Если лямбда обращается к полю/методу экземпляра без явного `this.`, компилятор неявно захватывает `this`. Для классов это просто ссылка — мутации через `this` внутри лямбды видны снаружи. Для **структур** это ловушка: `this` в структуре захватывается **по значению** (копируется), поэтому изменения полей структуры внутри лямбды **не влияют** на исходный экземпляр:

```csharp
struct Counter
{
    public int Count;
    public Action GetIncrementer() => () => Count++;  // CS1673 или неожиданное поведение
}
```

Компилятор в таких случаях либо выдаёт ошибку CS1673 ("Anonymous methods, lambda expressions... inside structs cannot access instance members of 'this'... consider copying to a local variable"), либо (в зависимости от контекста) молча работает с копией. Рекомендация: извлечь нужные значения полей в локальные переменные перед лямбдой, либо использовать локальную функцию, которая может обращаться к `this` напрямую по ссылке.

**Ресурсы.**
- [Errors and warnings — CS1673](https://learn.microsoft.com/dotnet/csharp/language-reference/compiler-messages/lambda-expression-errors#syntax-limitations-in-lambda-expressions)

---

### Вопрос 3.6 🟡 Что такое `static` лямбда и зачем она нужна?

**Ответ.** Модификатор `static` (с C# 9) явно запрещает лямбде захватывать локальные переменные, параметры или `this` из окружающего кода:

```csharp
Func<int,bool> isEven = static value => value % 2 == 0;   // OK
int factor = 2;
Func<int,int> bad = static x => x * factor;               // CS8820: ошибка компиляции
```

Смысл — гарантировать отсутствие замыкания и, соответственно, отсутствие лишней аллокации объекта-замыкания на каждый вызов метода, создающего лямбду. Это полезно в горячих путях кода, где нужно исключить скрытые аллокации, и как явная документация намерения ("эта лямбда не имеет состояния снаружи"). Статическая лямбда всё ещё может обращаться к статическим полям/методам и константам.

**Ресурсы.**
- [Static anonymous functions](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/lambda-expressions#static-lambdas)
- [CS8820 / CS8821](https://learn.microsoft.com/dotnet/csharp/language-reference/compiler-messages/lambda-expression-errors#syntax-limitations-in-lambda-expressions)

---

### Вопрос 3.7 🔴 Как замыкание влияет на количество и группировку heap-аллокаций, если лямбда захватывает переменные из разных вложенных областей видимости?

**Ответ.** Компилятор Roslyn старается минимизировать число объектов-замыканий: если несколько лямбд в одном методе захватывают переменные из одной и той же области видимости (scope), они могут получить **один общий** класс-замыкание с несколькими полями. Но если переменные объявлены во **вложенных** блоках (например, внутри `if` или цикла) и захватываются разными лямбдами на разных уровнях вложенности, компилятор создаёт **цепочку** классов-замыканий, где внутренний класс хранит ссылку на внешний (аналог nested scope в JS):

```csharp
void M()
{
    int outer = 1;
    for (int i = 0; i < 3; i++)     // "for" создаёт свою область на каждую итерацию для i, если i захвачен
    {
        int inner = i * 2;
        Action a = () => Console.WriteLine(outer + inner);
        // компилятор: класс для {outer}, вложенный класс для {inner}, ссылающийся на первый
    }
}
```

Практический вывод: чем "теснее" объявлена переменная к точке использования в лямбде, тем меньше объём замыкания и тем более гранулярны аллокации. Для кода в горячем пути стоит явно проверять сгенерированный IL/использовать sharplab.io, чтобы понять, сколько объектов-замыканий реально создаётся.

**Ресурсы.**
- [Local functions — heap allocations](https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/local-functions#local-functions-vs-lambda-expressions)
- Eric Lippert, блог о деталях реализации замыканий в Roslyn (серия "Closures examined")

---

### Вопрос 3.8 🟡 Может ли лямбда захватывать `IDisposable`-объект, и какие с этим связаны риски?

**Ответ.** Да, и это источник ошибок `ObjectDisposedException`. Если объект захвачен по ссылке (а не значение скопировано), а метод-владелец вызывает `Dispose()` до того, как выполнится делегат (или до того, как скомпилированное из expression tree выражение вызывается), обращение к объекту внутри лямбды упадёт:

```csharp
private static Func<int,int> CreateBoundResource()
{
    using var res = new Resource();   // implements IDisposable
    Expression<Func<int,int>> expr = b => res.Argument + b;
    var func = expr.Compile();
    return func;   // res уже Dispose() к моменту возврата
}
// func(1) выбросит ObjectDisposedException при обращении к res.Argument
```

Правило: не захватывать `IDisposable`-объекты, чей disposal-scope может закончиться раньше, чем закончится жизнь делегата/скомпилированного выражения. Либо материализовать нужные данные (например, `res.Argument`) в обычную переменную заранее, до `Dispose()`.

**Ресурсы.**
- [Execute expression trees — Caveats](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-execution#caveats)

---

### Вопрос 3.9 🟢 Видны ли переменные, объявленные внутри лямбды, снаружи неё?

**Ответ.** Нет. Переменные, введённые внутри тела лямбды, имеют область видимости, ограниченную телом лямбды, — они не видны в объемлющем методе. Это симметрично обычным правилам области видимости блоков в C#. Обратное — переменные снаружи — видны внутри лямбды (это и есть захват), если только явно не отгорожены модификатором `static` у самой лямбды.

**Ресурсы.**
- [Capture of outer variables — rule 2](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/lambda-expressions#capture-of-outer-variables-and-variable-scope-in-lambda-expressions)

---

### Вопрос 3.10 🟡 Как definite assignment (правило обязательного присваивания) работает с переменными, захватываемыми лямбдой?

**Ответ.** Компилятор требует, чтобы захваченная переменная была **точно присвоена** (definitely assigned) до её использования внутри лямбды — так же, как и в обычном коде. Но для локальных функций (в отличие от лямбд) компилятор может провести более тонкий статический анализ и признать переменную присвоенной, даже если присваивание происходит **внутри** самой локальной функции, если вызов этой функции гарантированно происходит после присваивания:

```csharp
int M()
{
    int y;
    LocalFunction();
    return y;                 // y считается точно присвоенной здесь

    void LocalFunction() => y = 0;
}
```

Для лямбд такой анализ работать не будет — переменную нужно инициализировать явно перед объявлением лямбды.

**Ресурсы.**
- [Local functions — variable capture](https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/local-functions#local-functions-vs-lambda-expressions)

---

## 4. Expression-bodied members

### Вопрос 4.1 🟢 Что такое expression-bodied member и какой синтаксис у него?

**Ответ.** Это сокращённая форма записи члена класса (метода, свойства, оператора, конструктора и т.д.), тело которого сводится к одному выражению. Синтаксис — `=>` после сигнатуры члена вместо блока `{ }`:

```csharp
public override string ToString() => $"{fname} {lname}".Trim();

// эквивалентно:
public override string ToString()
{
    return $"{fname} {lname}".Trim();
}
```

Не путать с лямбда-оператором `=>` — здесь тот же токен используется как **разделитель имени члена и его реализации** (expression body definition), а не для создания анонимной функции. Появилась в C# 6 (методы, операторы, read-only свойства/индексаторы), расширена в C# 7 (конструкторы, финализаторы, аксессоры свойств/событий).

**Ресурсы.**
- [Expression body definitions — lambda operator](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/lambda-operator#expression-body-definition)

---

### Вопрос 4.2 🟢 Для каких членов класса можно использовать expression-bodied синтаксис?

**Ответ.** Полный список (C# 7+):
1. **Методы и локальные функции**: `T M() => expr;` / `void M() => statementExpr;`
2. **Операторы**: `public static T operator +(T a, T b) => expr;`
3. **Свойства и индексаторы** (только read-only, форма целиком `T P => expr;`) либо отдельные аксессоры: `get => expr; set => stmt; init => stmt;`
4. **Конструкторы и финализаторы**: `C() => stmt;` / `~C() => stmt;`
5. **Аксессоры событий**: `add => stmt; remove => stmt;`

```csharp
public class Person
{
    public string Name => $"{First} {Last}";      // property
    public Person(string f, string l) => (First, Last) = (f, l);   // constructor + tuple assignment
    public static Person operator +(Person a, Person b) => Merge(a, b);  // operator
    ~Person() => Cleanup();                        // finalizer
}
```

**Ресурсы.**
- [Expression body definitions](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/lambda-operator#expression-body-definition)

---

### Вопрос 4.3 🟢 Может ли expression-bodied свойство иметь `set`-аксессор?

**Ответ.** В форме `T P => expr;` — нет, эта краткая форма всегда read-only (эквивалент только `get`). Но каждый аксессор можно писать в expression-bodied форме по отдельности внутри обычного блочного свойства:

```csharp
public string Name
{
    get => _name;
    set => _name = value ?? throw new ArgumentNullException(nameof(value));
}
```

Для `void`-члена, `async`-метода, конструктора и аксессоров `set`/`init`/`add`/`remove` тело должно быть *statement expression* (присваивание, вызов метода, `new`, инкремент/декремент, `await`) — то есть выражением, которое может стоять как самостоятельный оператор.

**Ресурсы.**
- [Expression body definitions — properties and indexers](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/lambda-operator#expression-body-definition)

---

### Вопрос 4.4 🟡 В чём разница между `person.Name => $"{First} {Last}"` (вычисляемое каждый раз свойство) и полем, инициализированным один раз?

**Ответ.** Expression-bodied свойство **пересчитывается на каждое обращение** — оно не кэширует значение:

```csharp
public double Distance => Math.Sqrt(X * X + Y * Y);   // вычисляется заново при каждом чтении
```

Это критично для типов, где значение свойства должно отражать *текущее* состояние объекта (особенно для record-типов, где `with`-выражение меняет свойства). Если закэшировать в инициализаторе (`public double Distance { get; } = Math.Sqrt(...)`), значение "заморозится" на момент создания и будет некорректным после `with`-копирования с изменёнными базовыми полями — классическая ошибка при проектировании computed properties в records.

**Ресурсы.**
- [Records — computed properties should be computed on access](https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/record#nondestructive-mutation)

---

### Вопрос 4.5 🟢 Как правило стиля IDE (IDE0022, IDE0053 и др.) регулирует использование expression-bodied синтаксиса?

**Ответ.** .NET code-style анализатор предоставляет отдельные правила для разных категорий членов: `IDE0021` (конструкторы), `IDE0022` (методы), `IDE0023`/`IDE0024` (операторы), `IDE0025` (свойства), `IDE0053` (лямбды!). Каждое поддерживает три значения опции: `true` (всегда предпочитать expression body), `false` (всегда блок), `when_on_single_line` (только если тело умещается в одну строку). Это настраивается в `.editorconfig`:

```ini
csharp_style_expression_bodied_methods = when_on_single_line
csharp_style_expression_bodied_lambdas = true
```

Интересный факт: для лямбд (`IDE0053`) значение по умолчанию — `true` (предпочитать expression body), в отличие от методов, где по умолчанию `false`.

**Ресурсы.**
- [Use expression body for methods (IDE0022)](https://learn.microsoft.com/dotnet/fundamentals/code-analysis/style-rules/ide0022)
- [Use expression body for lambdas (IDE0053)](https://learn.microsoft.com/dotnet/fundamentals/code-analysis/style-rules/ide0053)

---

### Вопрос 4.6 🟡 Компилируются ли expression-bodied члены во что-то отличное от обычных методов на уровне IL?

**Ответ.** Нет — это чисто синтаксический сахар на уровне компилятора C#. `public int GetAge() => Age;` и `public int GetAge() { return Age; }` порождают идентичный (или практически идентичный) IL. Никакого рантайм-отличия, накладных расходов или иной семантики выполнения — выбор формы влияет только на читаемость исходного кода, а не на производительность или поведение.

**Ресурсы.**
- [Methods in C# — Expression-bodied members](https://learn.microsoft.com/dotnet/csharp/methods#expression-bodied-members)

---

### Вопрос 4.7 🟡 Как связаны expression-bodied конструкторы с деконструкцией через tuple-присваивание?

**Ответ.** Частый идиоматичный паттерн — присвоить сразу несколько полей через tuple-deconstruction в теле expression-bodied конструктора:

```csharp
public class Point
{
    public int X { get; }
    public int Y { get; }
    public Point(int x, int y) => (X, Y) = (x, y);
}
```

Это работает, потому что присваивание tuple `(X, Y) = (x, y)` — это единственное statement-expression (компилируется в последовательность из двух присваиваний полям), что удовлетворяет требованию "тело конструктора — statement expression". Такой стиль особенно распространён в records и DTO с большим числом полей, где multi-line блок с `this.X = x; this.Y = y;` выглядит избыточно многословным.

**Ресурсы.**
- [Deconstructing tuples and other types](https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/value-tuples#deconstruction)

---

## 5. LINQ и лямбды

### Вопрос 5.1 🟢 Как лямбды используются в стандартных операторах LINQ (`Where`, `Select`, `OrderBy`)?

**Ответ.** Большинство операторов LINQ to Objects принимают делегат `Func<TSource,...>`, куда естественно подставляется лямбда:

```csharp
int[] numbers = { 5, 4, 1, 3, 9, 8, 6, 7, 2, 0 };
var result = numbers
    .Where(n => n % 2 == 1)          // Func<int,bool> — предикат фильтрации
    .Select(n => n * n)              // Func<int,int> — проекция
    .OrderByDescending(n => n)       // Func<int,int> — ключ сортировки
    .ToList();
```

`Where` принимает предикат (`Func<T,bool>`), `Select` — проекцию (`Func<T,TResult>`), `OrderBy`/`GroupBy` — селектор ключа. Существуют перегрузки, дополнительно передающие индекс элемента: `Where((item, index) => ...)`.

**Ресурсы.**
- [Enumerable.Where](https://learn.microsoft.com/dotnet/api/system.linq.enumerable.where)
- [Lambdas with the standard query operators](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/lambda-expressions#lambdas-with-the-standard-query-operators)

---

### Вопрос 5.2 🟢 В чём разница между method syntax и query syntax в LINQ, и как лямбды относятся к каждой?

**Ответ.** *Method syntax* — цепочка вызовов методов расширения с лямбдами: `numbers.Where(n => n > 0).Select(n => n * 2)`. *Query syntax* — SQL-подобный синтаксис: `from n in numbers where n > 0 select n * 2`. Компилятор транслирует query syntax в эквивалентные вызовы method syntax **на этапе компиляции** — то есть под капотом всё равно генерируются лямбды и вызовы `Where`/`Select`. Не все операторы (`Skip`, `Take`, `Aggregate` и др.) имеют query-syntax представление — для них приходится смешивать оба стиля или использовать чисто method syntax.

**Ресурсы.**
- [LINQ queries](https://learn.microsoft.com/dotnet/csharp/fundamentals/statements/linq)

---

### Вопрос 5.3 🟡 Чем `Select` отличается от `SelectMany`, и как лямбда меняется между ними?

**Ответ.** `Select` применяет проекцию `Func<T,TResult>` к каждому элементу, возвращая последовательность из одного результата на элемент (1:1). `SelectMany` применяет проекцию `Func<T,IEnumerable<TResult>>`, возвращая **плоскую** последовательность, "расплющивая" вложенные коллекции (1:N):

```csharp
var orders = customers.Select(c => c.Orders);       // IEnumerable<IEnumerable<Order>> — вложенность!
var flatOrders = customers.SelectMany(c => c.Orders); // IEnumerable<Order> — плоский список
```

`SelectMany` также поддерживает перегрузку с двумя лямбдами — селектором коллекции и селектором результата, что эквивалентно `join`/вложенному `from` в query syntax:

```csharp
var pairs = customers.SelectMany(c => c.Orders, (c, o) => new { c.Name, o.Total });
```

**Ресурсы.**
- [Enumerable.SelectMany](https://learn.microsoft.com/dotnet/api/system.linq.enumerable.selectmany)

---

### Вопрос 5.4 🟡 Как работает `Aggregate` и почему лямбда там принимает "накопитель"?

**Ответ.** `Aggregate` — это классический fold/reduce: лямбда `Func<TAccumulate, TSource, TAccumulate>` получает текущее накопленное значение и следующий элемент, возвращая новое накопленное значение:

```csharp
int[] numbers = { 4, 7, 10 };
int product = numbers.Aggregate(1, (acc, next) => acc * next); // 280
```

Первый параметр `Aggregate` — начальное значение (seed); без него используется первый элемент последовательности как начальный аккумулятор (и тогда для пустой последовательности выбрасывается исключение). Есть перегрузка с дополнительным `resultSelector`, применяемым к финальному аккумулятору.

**Ресурсы.**
- [Enumerable.Aggregate](https://learn.microsoft.com/dotnet/api/system.linq.enumerable.aggregate)

---

### Вопрос 5.5 🟢 Как передать индекс элемента в лямбду LINQ?

**Ответ.** У `Select` и `Where` (для `IEnumerable<T>`, но не для `IQueryable<T>` в большинстве провайдеров) есть перегрузки с индексом:

```csharp
var indexed = names.Select((name, index) => $"{index}: {name}");
var everySecond = numbers.Where((n, i) => i % 2 == 0);
```

Важно: провайдеры `IQueryable` (например, EF Core) обычно **не поддерживают** индексную перегрузку — попытка использовать `.Select((x, i) => ...)` в запросе к базе данных приведёт к исключению времени выполнения, так как SQL не может напрямую выразить "индекс в последовательности" без ORDER BY/ROW_NUMBER.

**Ресурсы.**
- [Enumerable.Select — overloads](https://learn.microsoft.com/dotnet/api/system.linq.enumerable.select)

---

### Вопрос 5.6 🟡 Что произойдёт, если лямбда в `Where`/`Select` бросит исключение — когда именно оно всплывёт?

**Ответ.** Из-за отложенного выполнения (см. раздел 6) исключение внутри лямбды всплывёт **не в момент вызова `Where`/`Select`**, а в момент реального перечисления (`foreach`, `ToList()`, `First()` и т.п.), и конкретно — в момент обработки того элемента, на котором лямбда упала:

```csharp
var query = numbers.Select(n => 100 / n);   // исключения ещё нет, даже если есть n == 0
var list = query.ToList();                  // DivideByZeroException бросится здесь, при материализации
```

Это частый источник confusion при отладке: стек вызовов исключения указывает на место материализации, а не на место объявления запроса.

**Ресурсы.**
- [Introduction to LINQ Queries — deferred execution](https://learn.microsoft.com/dotnet/csharp/linq/get-started/introduction-to-linq-queries#classification-of-standard-query-operators-by-manner-of-execution)

---

### Вопрос 5.7 🟡 Как работает `GroupBy` с лямбдой-селектором ключа и (опционально) селектором элемента?

**Ответ.**

```csharp
var groups = people.GroupBy(
    keySelector: p => p.Department,             // Func<Person,string>
    elementSelector: p => p.Name);               // Func<Person,string> — необязательный

foreach (IGrouping<string,string> g in groups)
    Console.WriteLine($"{g.Key}: {string.Join(", ", g)}");
```

`GroupBy` — **не потоковый** (nonstreaming) оператор: он должен прочитать весь источник, прежде чем вернуть первую группу, потому что группа не считается "завершённой", пока не просмотрены все элементы. Есть также перегрузка с `resultSelector`, сразу проецирующей каждую группу в нужный тип, без промежуточного `IGrouping<TKey,TElement>`.

**Ресурсы.**
- [Enumerable.GroupBy](https://learn.microsoft.com/dotnet/api/system.linq.enumerable.groupby)

---

### Вопрос 5.8 🟡 В чём разница между `First`, `FirstOrDefault`, `Single`, `SingleOrDefault` с точки зрения лямбды-предиката и обработки ошибок?

**Ответ.**
- `First(predicate)` — возвращает первый подходящий элемент, бросает `InvalidOperationException`, если ни один не подходит.
- `FirstOrDefault(predicate)` — то же, но возвращает `default(T)` вместо исключения.
- `Single(predicate)` — требует **ровно один** подходящий элемент; бросает исключение и если 0, и если больше 1 совпадений.
- `SingleOrDefault(predicate)` — `default(T)` при 0 совпадений, исключение при >1.

Все четыре — операторы с **немедленным** (immediate) выполнением: они сами инициируют перечисление источника и завершают его, как только условие удовлетворено (для `First*`) или сразу выполняют полный проход, если нужно проверить уникальность (`Single*`).

**Ресурсы.**
- [Enumerable.First](https://learn.microsoft.com/dotnet/api/system.linq.enumerable.first)
- [Enumerable.Single](https://learn.microsoft.com/dotnet/api/system.linq.enumerable.single)

---

### Вопрос 5.9 🟡 Почему многократное перечисление одного и того же LINQ-запроса — плохая практика (multiple enumeration)?

**Ответ.** Так как большинство операторов используют отложенное выполнение, каждое повторное перечисление одного и того же `IEnumerable<T>`-запроса **повторно выполняет всю цепочку** — включая повторное чтение источника (файл, БД-запрос через `IQueryable`, дорогая лямбда с побочными эффектами):

```csharp
IEnumerable<int> query = data.Where(x => IsExpensiveCheck(x));
int count = query.Count();       // проход №1: IsExpensiveCheck вызывается для всех элементов
var list = query.ToList();       // проход №2: IsExpensiveCheck вызывается СНОВА для всех элементов
```

Анализатор Roslyn (`CA1851`, а также многие статические анализаторы вроде ReSharper) предупреждает про "Possible multiple enumeration of IEnumerable". Решение — материализовать результат один раз (`.ToList()`/`.ToArray()`), если он используется многократно.

**Ресурсы.**
- [CA1851: Possible multiple enumerations of IEnumerable collection](https://learn.microsoft.com/dotnet/fundamentals/code-analysis/quality-rules/ca1851)

---

### Вопрос 5.10 🟢 Как замыкание в LINQ-запросе взаимодействует с изменяющейся внешней переменной?

**Ответ.** Так как `Where`/`Select` откладывают выполнение, а лямбда захватывает переменную по ссылке, изменение переменной **после** объявления запроса, но **до** его перечисления, повлияет на результат:

```csharp
int threshold = 5;
var query = numbers.Where(n => n > threshold);
threshold = 100;
var result = query.ToList();   // используется threshold == 100, а не 5!
```

Это то же самое поведение замыкания, что и в разделе 3, но применительно к LINQ — классический вопрос "что выведет код" на собеседовании.

**Ресурсы.**
- [Capture of outer variables](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/lambda-expressions#capture-of-outer-variables-and-variable-scope-in-lambda-expressions)

---

### Вопрос 5.11 🟡 Как реализовать собственный LINQ-оператор расширения с отложенным выполнением, использующим `yield return` и принимающим лямбду?

**Ответ.**

```csharp
public static IEnumerable<TResult> MySelect<TSource, TResult>(
    this IEnumerable<TSource> source, Func<TSource, TResult> selector)
{
    foreach (var item in source)
        yield return selector(item);
}
```

Благодаря `yield return`, метод компилируется в машину состояний (state machine), реализующую `IEnumerable<T>`/`IEnumerator<T>`, и тело метода **не выполняется**, пока не начнётся перечисление — это и обеспечивает отложенное выполнение, аналогичное встроенным операторам LINQ. Важная деталь: даже проверка аргументов на `null` в таком методе с `yield return` **откладывается** — она выполнится не при вызове `MySelect(...)`, а при первом `MoveNext()`. Чтобы проверять аргументы сразу (fail-fast), используют паттерн с обёрткой: публичный non-yield метод валидирует аргументы и вызывает приватный итератор.

**Ресурсы.**
- [Iterators (C#)](https://learn.microsoft.com/dotnet/csharp/programming-guide/concepts/iterators)

---

### Вопрос 5.12 🟢 Чем `Any(predicate)` лучше, чем `Count(predicate) > 0`?

**Ответ.** `Any(predicate)` — потоковый оператор, который останавливает перечисление, как только найден первый подходящий элемент (short-circuit). `Count(predicate)` вынужден пройти **всю** последовательность, чтобы посчитать точное число совпадений, даже если нужен только факт "есть хотя бы один". Для больших или бесконечных последовательностей (генераторов через `yield`) `Count(predicate) > 0` может быть значительно медленнее или вовсе не завершиться, тогда как `Any(predicate)` завершится сразу после первого совпадения.

**Ресурсы.**
- [Enumerable.Any](https://learn.microsoft.com/dotnet/api/system.linq.enumerable.any)

---

## 6. Отложенное выполнение: IEnumerable vs IQueryable

### Вопрос 6.1 🟢 Что такое отложенное (deferred) выполнение в LINQ?

**Ответ.** Отложенное выполнение означает, что операция, описанная в запросе, **не выполняется в момент объявления** запроса — она выполняется только тогда, когда результат реально запрашивается (перечисление через `foreach`, вызов `ToList()`, `Count()` и т.п.). Почти все операторы, возвращающие `IEnumerable<T>` (`Where`, `Select`, `OrderBy`, `GroupBy`), — отложенные; операторы, возвращающие единичное скалярное значение (`Count()`, `Sum()`, `First()`, `ToList()`, `ToArray()`) — выполняются немедленно (immediate/eager execution).

```csharp
var query = numbers.Where(n => n > 0);   // ничего ещё не выполнено
numbers.Add(-5);                          // источник изменился ДО перечисления
foreach (var n in query) { ... }          // выполняется здесь, видит уже изменённый источник
```

**Диаграмма (жизненный цикл LINQ-запроса).**
```
Объявление запроса         Изменение источника        Перечисление (foreach/.ToList())
        │                          │                              │
        ▼                          ▼                              ▼
 var q = src.Where(...)   →   src.Add(x)   →   foreach (var i in q) { ... }
        │                                              │
   ничего не выполнено                     ЗДЕСЬ впервые выполняется вся цепочка
   (просто построено "дерево" операций)     операторов, читая ТЕКУЩЕЕ состояние src
```

**Ресурсы.**
- [Introduction to LINQ Queries — Classification of standard query operators](https://learn.microsoft.com/dotnet/csharp/linq/get-started/introduction-to-linq-queries#classification-of-standard-query-operators-by-manner-of-execution)
- [LINQ queries — Run a query](https://learn.microsoft.com/dotnet/csharp/fundamentals/statements/linq#run-a-query)

---

### Вопрос 6.2 🟡 В чём разница между streaming и non-streaming отложенными операторами?

**Ответ.** Оба типа откладывают выполнение, но различаются, сколько источника нужно прочитать, чтобы выдать **первый** элемент результата:
- **Streaming** (`Where`, `Select`, `Take`, `Skip`) — обрабатывают элемент источника сразу по мере чтения и могут выдать результат, прочитав лишь часть источника.
- **Non-streaming** (`OrderBy`, `GroupBy`, `Distinct`\*, `Reverse`) — обязаны прочитать **весь** источник, прежде чем выдать первый элемент результата, так как результат (например, отсортированный порядок) зависит от всех элементов сразу.

Практическое следствие: `source.OrderBy(x => x.Key).First()` всё равно требует прохода по всему `source`, даже если нужен только один элемент, — в отличие от `source.Where(...).First()`, который остановится на первом совпадении.

**Ресурсы.**
- [Classification table — streaming vs nonstreaming](https://learn.microsoft.com/dotnet/csharp/linq/get-started/introduction-to-linq-queries#classification-table)

---

### Вопрос 6.3 🟡 Чем принципиально отличается `IQueryable<T>` от `IEnumerable<T>` в контексте лямбд?

**Ответ.** `IEnumerable<T>` работает с лямбдами, скомпилированными в **делегаты** (`Func<>`) — код лямбды исполняется прямо в CLR, элемент за элементом (LINQ to Objects). `IQueryable<T>` работает с лямбдами, скомпилированными в **expression trees** (`Expression<Func<>>`) — лямбда передаётся как *данные*, описывающие намерение, а конкретный провайдер (EF Core, LINQ to SQL) **транслирует** это дерево в другой язык запросов (обычно SQL) и выполняет запрос там, а не в CLR.

```csharp
IEnumerable<Customer> a = dbSet.AsEnumerable().Where(c => c.Age > 18);  // фильтрация в памяти после выгрузки ВСЕХ строк
IQueryable<Customer>  b = dbSet.Where(c => c.Age > 18);                  // фильтрация переведена в SQL WHERE, на сервере БД
```

Это одна из самых частых тем на собеседованиях — путаница между `AsEnumerable()`/`ToList()` слишком рано в цепочке запроса и результирующим падением производительности (вся таблица выгружается в память до фильтрации).

**Ресурсы.**
- [Queryable.Select](https://learn.microsoft.com/dotnet/api/system.linq.queryable.select)
- [Lambdas with the standard query operators](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/lambda-expressions#lambdas-with-the-standard-query-operators)

---

### Вопрос 6.4 🔴 Как компилятор решает, компилировать ли лямбду в делегат или в Expression Tree при вызове перегруженного метода расширения (`Where` для `IEnumerable` vs `Queryable`)?

**Ответ.** Решение принимается на этапе **разрешения перегрузки** (overload resolution) на основе **статического типа** переменной, к которой применяется метод, а не runtime-типа объекта:

```csharp
IQueryable<Customer> q = dbSet;                 // статический тип — IQueryable<T>
var filtered = q.Where(c => c.Age > 18);
// вызывается Queryable.Where(this IQueryable<T>, Expression<Func<T,bool>>)
// => лямбда компилируется в EXPRESSION TREE, а не в делегат
```

Если бы `q` был объявлен как `IEnumerable<Customer>`, вызвался бы `Enumerable.Where(this IEnumerable<T>, Func<T,bool>)`, и та же самая по тексту лямбда скомпилировалась бы в обычный **делегат**. Именно поэтому важно не терять статический тип `IQueryable<T>` по пути (например, случайным `.AsEnumerable()` или присвоением переменной с неверным типом) — иначе фильтрация "утечёт" из SQL в память.

**Ресурсы.**
- [Lambda expressions — same lambda, different underlying type](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/lambda-expressions#lambdas-with-the-standard-query-operators)

---

### Вопрос 6.5 🟡 Почему вызов `.ToList()` в середине цепочки `IQueryable`-запроса — частая ошибка производительности?

**Ответ.** `.ToList()` немедленно материализует запрос **в его текущем виде**, что означает выполнение SQL-запроса прямо сейчас, с загрузкой всех выбранных на этот момент строк в память. Если дальнейшая фильтрация/сортировка добавляется **после** `.ToList()`, она выполняется уже над `List<T>` в памяти через LINQ to Objects, а не транслируется в SQL:

```csharp
var result = dbContext.Orders
    .Where(o => o.CustomerId == id)   // ещё SQL
    .ToList()                          // ЗДЕСЬ выполняется SQL-запрос и грузятся строки в память
    .Where(o => o.Total > 100)         // это уже LINQ to Objects — фильтрация в памяти!
    .OrderBy(o => o.Date);
```

Если исходная таблица `Orders` большая, а нужен маленький итоговый набор, такой код тянет из БД гораздо больше данных, чем нужно. Правило: держать `.ToList()`/`.ToArray()` в самом конце цепочки, когда все фильтры/проекции уже применены к `IQueryable`.

**Ресурсы.**
- [Language Integrated Query (LINQ) — remote data](https://learn.microsoft.com/dotnet/csharp/linq/#how-to-enable-linq-querying-of-your-data-source)

---

### Вопрос 6.6 🟡 Как ленивая (`lazy`) и жадная (`eager`) оценка соотносятся с deferred execution?

**Ответ.** Deferred execution касается *момента* выполнения (сейчас vs позже); lazy/eager evaluation касается *того, как много обрабатывается за один шаг перечисления*, когда выполнение всё же откладывается:
- **Lazy evaluation** — при каждом вызове `MoveNext()` обрабатывается ровно один элемент источника (типично для итераторов на `yield return`, `Where`, `Select`).
- **Eager evaluation** внутри отложенного оператора — при **первом же** вызове `MoveNext()` обрабатывается **вся** коллекция целиком (например, `OrderBy` должен отсортировать всё, прежде чем отдать первый элемент), при этом сам оператор всё ещё "отложенный" в смысле общего времени срабатывания.

Итого: отложенное выполнение — не гарантия построчной (streaming) обработки; non-streaming операторы всё ещё откладывают момент запуска, но при запуске работают "жадно" по объёму.

**Ресурсы.**
- [Deferred execution and lazy evaluation (LINQ to XML)](https://learn.microsoft.com/dotnet/standard/linq/deferred-execution-lazy-evaluation)

---

### Вопрос 6.7 🟡 Что произойдёт, если внутри `Where`, применяемого к `IQueryable`, вызвать обычный C#-метод, которого нет в поддерживаемых SQL-провайдером трансляциях?

**Ответ.** Провайдер (например, EF Core) попытается транслировать expression tree в SQL. Если встречает вызов метода, для которого нет известного соответствия (например, кастомный C#-метод с произвольной логикой), поведение зависит от версии провайдера:
- Старые версии EF (6.x, EF Core до 3.0) могли молча выполнять часть запроса на клиенте (client evaluation) — источник тонких багов и деградации производительности.
- Начиная с EF Core 3.0, при невозможности трансляции метода в SQL выбрасывается `InvalidOperationException` во время выполнения запроса — явная ошибка вместо тихого fallback.

Некоторые простые методы (`string.StartsWith`, `string.Contains`, `Math.Abs`) провайдеры **умеют** транслировать в эквивалентные SQL-конструкции (`LIKE`, встроенные функции). Пользовательские методы — как правило, нет, если только это не extension-метод, специально зарегистрированный провайдером для трансляции (`EF.Functions.Like` и т.п.).

**Ресурсы.**
- [Expression lambdas — query provider translation](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/lambda-expressions#expression-lambdas)

---

### Вопрос 6.8 🟢 Как принудительно материализовать (force) выполнение отложенного запроса?

**Ответ.** Любой из следующих вызовов немедленно выполняет всю цепочку и возвращает конкретную коллекцию/значение вместо описания запроса: `.ToList()`, `.ToArray()`, `.ToDictionary()`, `.ToHashSet()`, `.ToLookup()`, а также скалярные операторы (`Count()`, `Sum()`, `First()`, `Any()`, `Max()` и т.д.) и явное перечисление через `foreach`. Выбор конкретного метода материализации зависит от того, что нужно дальше: `List<T>` для индексируемого доступа и повторной итерации, `HashSet<T>` для быстрой проверки на вхождение, `Dictionary<TKey,TValue>` для быстрого поиска по ключу.

**Ресурсы.**
- [Enumerable.ToList](https://learn.microsoft.com/dotnet/api/system.linq.enumerable.tolist)

---

## 7. Expression Trees

### Вопрос 7.1 🟡 Что такое Expression Tree и чем `Expression<Func<T>>` отличается от `Func<T>`?

**Ответ.** Expression Tree — представление кода **как структуры данных** (дерева объектов), а не как исполняемых инструкций. `Func<T>` — это делегат: скомпилированный исполняемый код, готовый к вызову. `Expression<Func<T>>` — это дерево объектов `Expression`, *описывающее* ту же логику, которое можно анализировать, изменять и (при необходимости) скомпилировать в делегат через `.Compile()`:

```csharp
Func<int,int> del = x => x + 1;              // Код — готов к вызову: del(5) → 6
Expression<Func<int,int>> exp = x => x + 1;  // Данные — дерево, описывающее "x + 1"
Func<int,int> del2 = exp.Compile();          // теперь можно вызвать: del2(5) → 6
```

Компилятор порождает совершенно разный код для этих двух объявлений при абсолютно одинаковом синтаксисе лямбды — выбор зависит от **типа переменной слева**. Expression trees лежат в основе `IQueryable`-провайдеров (EF Core, LINQ to SQL), позволяя транслировать C#-код в SQL, а также используются в динамическом построении правил, ORM-мапперах, сериализаторах выражений.

**Диаграмма.**
```
x => x + 1
        │
        ▼
┌───────────────────────────────┐
│ LambdaExpression               │
│  Parameters: [ x : int ]       │
│  Body: BinaryExpression (Add)  │
│          ├── Left:  ParameterExpression "x"
│          └── Right: ConstantExpression 1
└───────────────────────────────┘
```

**Ресурсы.**
- [Expression Trees (overview)](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/)
- [8.6 Expression tree types — C# language specification](https://learn.microsoft.com/dotnet/csharp/language-reference/language-specification/types#86-expression-tree-types)

---

### Вопрос 7.2 🟢 Как скомпилировать Expression Tree обратно в исполняемый делегат?

**Ответ.** Метод `Compile()`, определённый на `LambdaExpression` (и, следовательно, на `Expression<TDelegate>`), генерирует IL "на лету" (через `System.Reflection.Emit` под капотом) и возвращает делегат нужного типа:

```csharp
Expression<Func<int,bool>> lessThanFive = num => num < 5;
Func<int,bool> compiled = lessThanFive.Compile();
Console.WriteLine(compiled(3));  // True
```

Только `LambdaExpression`/`Expression<TDelegate>` можно скомпилировать в исполняемый код — остальные типы узлов (`ConstantExpression`, `BinaryExpression` и т.п.) сами по себе не имеют смысла "выполнить их напрямую"; они существуют только как часть дерева, венчающегося `LambdaExpression`.

**Ресурсы.**
- [Execute expression trees](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-execution)

---

### Вопрос 7.3 🟡 Какие ограничения накладываются на лямбды, из которых компилятор может построить Expression Tree?

**Ответ.** Компилятор C# строит expression tree только из **expression lambda** (не statement lambda), причём далеко не любое выражение допустимо. Запрещены, среди прочего:
- `statement`-конструкции (присваивания, циклы, `if`/`else`-операторы, `try/catch`) — так как дерево описывает *выражение*, а не последовательность операторов;
- `await`/`async`-лямбды;
- ссылки на локальные функции;
- обращение к `base`;
- `dynamic`-операции;
- null-coalescing/null-conditional операторы с `null`/`default` слева, `??=`, `?.` (в более старых версиях — полностью запрещены; сейчас — с оговорками в зависимости от версии .NET);
- `unsafe`-код и указатели.

```csharp
Expression<Func<int>> bad1 = () => { return 1; };     // ошибка: statement lambda
Expression<Action>    bad2 = async () => await Task.Delay(1); // ошибка: async
```

Причина ограничений — многие такие конструкции не имеют естественного соответствия в модели "дерево выражений", либо (в случае новых фич C# 6+) добавление новых видов узлов было бы breaking change для существующих интерпретаторов деревьев (Entity Framework и т.п.), поэтому такие фичи "разворачиваются" в более старый эквивалентный синтаксис или не поддерживаются вовсе.

**Ресурсы.**
- [Expression Trees — Limitations](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/#limitations)

---

### Вопрос 7.4 🔴 Как построить Expression Tree вручную, без использования лямбды (через API `System.Linq.Expressions.Expression`)?

**Ответ.** Expression trees — иммутабельны, поэтому строятся "снизу вверх": сначала листья (константы, параметры), затем узлы, комбинирующие их. Пример построения дерева для `() => 1 + 2`:

```csharp
var one = Expression.Constant(1, typeof(int));
var two = Expression.Constant(2, typeof(int));
var addition = Expression.Add(one, two);
var lambda = Expression.Lambda(addition);          // без параметров
Func<int> compiled = (Func<int>)lambda.Compile();
Console.WriteLine(compiled());                       // 3
```

Более сложный пример с параметрами и вызовом метода — дерево для `(x, y) => Math.Sqrt(x * x + y * y)`:

```csharp
var xParam = Expression.Parameter(typeof(double), "x");
var yParam = Expression.Parameter(typeof(double), "y");
var xSquared = Expression.Multiply(xParam, xParam);
var ySquared = Expression.Multiply(yParam, yParam);
var sum = Expression.Add(xSquared, ySquared);
var sqrtMethod = typeof(Math).GetMethod("Sqrt", new[] { typeof(double) })!;
var distanceCall = Expression.Call(sqrtMethod, sum);
var lambda = Expression.Lambda<Func<double,double,double>>(distanceCall, xParam, yParam);

Func<double,double,double> distance = lambda.Compile();
Console.WriteLine(distance(3, 4));   // 5
```

Такой ручной API открывает то, что недоступно из обычных лямбд: например, узлы условной логики (`Expression.Condition`), циклов (`Expression.Loop`), блоков с несколькими операторами (`Expression.Block`) и даже `try/catch` (`Expression.TryCatch`) — то есть можно построить дерево, эквивалентное statement lambda, минуя ограничение компилятора C#.

**Диаграмма (дерево для `Math.Sqrt(x*x + y*y)`).**
```
                 LambdaExpression
                 Params: [x, y]
                        │
                        ▼
                 MethodCallExpression (Math.Sqrt)
                        │
                        ▼
                 BinaryExpression (Add)
                 ┌──────┴──────┐
                 ▼             ▼
     BinaryExpression      BinaryExpression
     (Multiply: x*x)       (Multiply: y*y)
       ┌─────┴─────┐         ┌─────┴─────┐
       ▼           ▼         ▼           ▼
  Parameter"x" Parameter"x" Parameter"y" Parameter"y"
```

**Ресурсы.**
- [Build expression trees](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-building)
- [System.Linq.Expressions namespace](https://learn.microsoft.com/dotnet/api/system.linq.expressions)

---

### Вопрос 7.5 🔴 Как реализовать паттерн Visitor для обхода и модификации Expression Tree (`ExpressionVisitor`)?

**Ответ.** Так как деревья иммутабельны, чтобы "изменить" дерево, нужно построить **новое** дерево, копируя немодифицированные узлы и заменяя нужные. Базовый класс `ExpressionVisitor` реализует этот паттерн — переопределяются методы `VisitXxx` для интересующих типов узлов, а остальные узлы копируются автоматически базовой реализацией:

```csharp
public class ParameterReplacer : ExpressionVisitor
{
    private readonly ParameterExpression _oldParam;
    private readonly Expression _newExpr;

    public ParameterReplacer(ParameterExpression oldParam, Expression newExpr)
        => (_oldParam, _newExpr) = (oldParam, newExpr);

    protected override Expression VisitParameter(ParameterExpression node)
        => node == _oldParam ? _newExpr : base.VisitParameter(node);
}
```

Практический пример использования — динамическая композиция предикатов (`Expression<Func<T,bool>>`) через `AndAlso`/`OrElse`, где нужно "перепривязать" параметр одной лямбды к параметру другой, прежде чем их можно будет объединить в общее дерево (классическая библиотека для этого — `LinqKit`/`PredicateBuilder`). Также `ExpressionVisitor` используется всеми ORM-провайдерами для трансляции LINQ-выражений в SQL: они обходят дерево и генерируют текст запроса узел за узлом.

**Ресурсы.**
- [Interpret expressions (visitor for expression trees)](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-interpreting)
- [Translating expression trees](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-translating)

---

### Вопрос 7.6 🟡 Как узнать тип узла Expression Tree во время обхода (`NodeType`, `ExpressionType`)?

**Ответ.** Базовый класс `Expression` содержит свойство `NodeType` типа `ExpressionType` — перечисление всех возможных видов узлов (`Add`, `Constant`, `Parameter`, `Call`, `Conditional`, `Lambda`, `MemberAccess` и т.д.). Типичный паттерн — проверить `NodeType` (или использовать `is`/паттерн-матчинг по конкретному подтипу) и привести к нужному подтипу для доступа к специфичным свойствам:

```csharp
Expression<Func<int,int>> addFive = num => num + 5;
if (addFive.Body is BinaryExpression { NodeType: ExpressionType.Add } binExpr)
{
    Console.WriteLine(binExpr.Left);   // "num"
    Console.WriteLine(binExpr.Right);  // "5"
}
```

Это основа для написания собственных "интерпретаторов" выражений — рекурсивных функций, которые в зависимости от `NodeType` вызывают себя для дочерних узлов и комбинируют промежуточные результаты (аналог того, как это делают ORM для генерации SQL).

**Ресурсы.**
- [.NET Runtime support for expression trees](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-classes)
- [ExpressionType enum](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expressiontype)

---

### Вопрос 7.7 🟡 Почему нельзя написать рекурсивную функцию напрямую внутри `Expression<Func<T>>`, и как это обойти?

**Ответ.** Expression tree, построенное компилятором из лямбды, не может ссылаться само на себя (рекурсия), потому что на момент вычисления тела лямбды переменная, в которую его присваивают, ещё не инициализирована как `Expression`, — а даже если бы была, вызов "себя" внутри expression tree потребовал бы узла типа "вызов делегата/лямбды", который для expression tree, построенного компилятором из лямбда-выражения, недопустим:

```csharp
// Не работает: попытка вызвать "factorial" рекурсивно внутри Expression
Expression<Func<int,int>> factorial = n =>
    n == 0 ? 1 : n * factorial.Compile()(n - 1);   // тип несовместим / не то, что нужно
```

Обходной путь — сначала скомпилировать выражение в делегат, а внутри **уже скомпилированного** делегата обращаться к себе через `Func<>`-переменную (обычная рекурсия через делегат, как в разделе 2.10), либо строить дерево вручную через `Expression.Call`, явно указывая `MethodInfo` рекурсивного метода (а не пытаясь встроить саму лямбду).

**Ресурсы.**
- [Interpret expressions — Extending this sample (ограничения statement lambda и рекурсии)](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-interpreting#extending-this-sample)

---

### Вопрос 7.8 🟡 Как использовать Expression Trees для построения динамических предикатов (dynamic query building) в стиле "постепенно добавляемых фильтров"?

**Ответ.** Частый практический сценарий — UI с несколькими необязательными фильтрами, каждый из которых при наличии добавляет условие `AND` к общему запросу `IQueryable<T>`. Наивный подход — динамически комбинировать `Expression<Func<T,bool>>` через `Expression.AndAlso`, аккуратно перепривязывая параметр:

```csharp
Expression<Func<Product,bool>> BuildFilter(string? name, decimal? minPrice)
{
    Expression<Func<Product,bool>> predicate = p => true;

    if (name is not null)
    {
        Expression<Func<Product,bool>> byName = p => p.Name.Contains(name);
        predicate = CombineAnd(predicate, byName);
    }
    if (minPrice is not null)
    {
        Expression<Func<Product,bool>> byPrice = p => p.Price >= minPrice;
        predicate = CombineAnd(predicate, byPrice);
    }
    return predicate;
}
```

Функция `CombineAnd` должна использовать `ExpressionVisitor` (см. 7.5), чтобы заменить параметр второго предиката параметром первого перед объединением через `Expression.AndAlso`, иначе получится дерево с двумя разными `ParameterExpression`, что при компиляции даст неверный результат или исключение. Готовые библиотеки для этого — **LinqKit** (`PredicateBuilder`) и **System.Linq.Dynamic.Core**.

**Ресурсы.**
- [Build expression trees — map code constructs](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-building#map-code-constructs-to-expressions)

---

### Вопрос 7.9 🟡 Как выполнить факториал через Expression Trees, если нужна рекурсия/условная логика, а не просто прямое выражение?

**Ответ.** Простое присваивание лямбды `Expression<Func<int,int>>` не поддерживает циклы/рекурсию напрямую (см. 7.3, 7.7). Но через ручное построение дерева с `Expression.Loop`, `Expression.Label`, `Expression.Condition` можно эмулировать императивную логику:

```csharp
var value = Expression.Parameter(typeof(int), "value");
var result = Expression.Variable(typeof(int), "result");
var loopBreak = Expression.Label();

var body = Expression.Block(
    new[] { result },
    Expression.Assign(result, Expression.Constant(1)),
    Expression.Loop(
        Expression.IfThenElse(
            Expression.GreaterThan(value, Expression.Constant(1)),
            Expression.Block(
                Expression.Assign(result, Expression.Multiply(result, value)),
                Expression.PostDecrementAssign(value)),
            Expression.Break(loopBreak)),
        loopBreak),
    result);

var factorial = Expression.Lambda<Func<int,int>>(body, value).Compile();
Console.WriteLine(factorial(5));   // 120
```

Это демонстрирует, что API `System.Linq.Expressions` строго более выразителен, чем то, что компилятор C# способен породить из обычной лямбды, — он поддерживает произвольные управляющие конструкции.

**Ресурсы.**
- [Build expression trees — factorial example reference](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-building#map-code-constructs-to-expressions)

---

### Вопрос 7.10 🟡 Зачем в реальных проектах используют Expression Trees, помимо ORM/LINQ провайдеров?

**Ответ.** Практические сценарии:
- **Быстрый доступ к членам объекта через reflection без потери производительности** — построение и кэширование скомпилированного getter/setter через `Expression.Property`/`Expression.Lambda`, что в разы быстрее, чем `PropertyInfo.GetValue`/`SetValue` через чистый reflection на каждый вызов.
- **Валидация с человекочитаемыми сообщениями об ошибках** — библиотеки типа FluentValidation используют expression trees, чтобы извлечь имя свойства из `x => x.Email` для формирования сообщения "Email is required" без magic strings.
- **Сериализация запросов/правил** — превращение бизнес-правил в expression trees, которые можно сохранить, передать по сети (например, в виде AST) и выполнить в другом процессе.
- **AutoMapper и подобные мапперы** — построение проекций между DTO и доменными моделями во время выполнения.

**Ресурсы.**
- [.NET Runtime support for expression trees](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-classes)

---

### Вопрос 7.11 🔴 Насколько дорого вызывать `.Compile()` многократно, и как правильно кэшировать результат?

**Ответ.** `Compile()` — относительно дорогая операция: она генерирует IL "на лету" через reflection emit и JIT-компилирует его. Вызов `.Compile()` внутри горячего пути (например, на каждый HTTP-запрос или на каждый элемент коллекции) может стать серьёзным узким местом производительности. Правильный паттерн — скомпилировать **один раз** и закэшировать делегат (в статическом readonly поле, `Lazy<T>`, `ConcurrentDictionary<Type, Delegate>` для параметризованных по типу сценариев):

```csharp
private static readonly Func<Product,bool> _isExpensive =
    ((Expression<Func<Product,bool>>)(p => p.Price > 1000)).Compile();
```

Начиная с .NET 6/7, есть перегрузка `Compile(preferInterpretation: true)` (интерпретация дерева вместо полноценной генерации IL) — компромисс: быстрее "холодный старт" (нет затрат на emit/JIT), но медленнее сам вызов делегата, что может быть выгодно для выражений, которые выполняются очень редко.

**Ресурсы.**
- [LambdaExpression.Compile](https://learn.microsoft.com/dotnet/api/system.linq.expressions.lambdaexpression.compile)

---

### Вопрос 7.12 🟡 Как `Expression<TDelegate>` соотносится с системой типов C# — является ли она делегатом?

**Ответ.** Нет. `Expression<TDelegate>` — обычный generic-класс (`Expression<TDelegate> : LambdaExpression`), параметризованный типом делегата, который он *описывает*. Его нельзя вызвать напрямую как функцию (`exp(5)` не скомпилируется) — только через `exp.Compile()(5)`. Компилятор C# лишь предоставляет специальную неявную конверсию **из лямбда-выражения** в `Expression<TDelegate>` (генерируя код построения дерева), но сам класс `Expression<TDelegate>` — самое обычное runtime-представление данных, доступное и без всякого лямбда-синтаксиса (см. ручное построение, вопрос 7.4).

**Ресурсы.**
- [Expression<TDelegate> class](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expression-1)
- [8.6 Expression tree types](https://learn.microsoft.com/dotnet/csharp/language-reference/language-specification/types#86-expression-tree-types)

---

## 8. Func / Action / Predicate и пользовательские делегаты

### Вопрос 8.1 🟢 Сколько параметров поддерживают `Func<>` и `Action<>` "из коробки"?

**Ответ.** BCL определяет `Action` (0 параметров) и `Action<T1>` … `Action<T1..T16>` (до 16 входных параметров), а также `Func<TResult>` (0 параметров, только результат) и `Func<T1,TResult>` … `Func<T1..T16,TResult>`. Если нужно больше 16 параметров — это явный сигнал, что стоит сгруппировать параметры в объект (DTO/record) вместо того, чтобы городить собственный делегат с таким числом аргументов; это же общая рекомендация по читаемости API независимо от лимита BCL.

**Ресурсы.**
- [Func<> delegates family](https://learn.microsoft.com/dotnet/api/system.func-1)
- [Action<> delegates family](https://learn.microsoft.com/dotnet/api/system.action)

---

### Вопрос 8.2 🟡 Как работает `Func<T,TResult>` с точки зрения вариантности параметров типа (`in`/`out`)?

**Ответ.** `Func<in T, out TResult>` объявлен с **контравариантным** входным параметром (`in T`) и **ковариантным** параметром результата (`out TResult`). Это позволяет неявно приводить `Func<Derived, Base>` там, где ожидается `Func<Base, Derived>`... (см. подробное объяснение и пример в разделе 14, вопрос про вариантность).

**Ресурсы.**
- [Func<T,TResult> Delegate](https://learn.microsoft.com/dotnet/api/system.func-2)

---

### Вопрос 8.3 🟢 Когда стоит объявить собственный `delegate`, а не использовать `Func`/`Action`?

**Ответ.** Основные случаи:
1. Нужны параметры `ref`/`out`/`in` (не поддерживаются `Func`/`Action`).
2. Нужна выразительная семантика имени в публичном API (`ValidationHandler` понятнее, чем `Func<Order,ValidationResult>`, особенно если тип используется во многих местах).
3. Нужны значения параметров по умолчанию, специфичные для делегата (см. вопрос 1.3) — `Func`/`Action` не хранят собственные default-значения.
4. Совместимость со старым кодом/API, спроектированными до появления `Func`/`Action` в .NET 3.5.

Для большинства новых внутренних API рекомендация — использовать стандартные `Func`/`Action`, чтобы не плодить типы делегатов без необходимости и облегчить совместимость с LINQ/другими API, ожидающими стандартные делегаты.

**Ресурсы.**
- [Delegates (C# Programming Guide)](https://learn.microsoft.com/dotnet/csharp/programming-guide/delegates/)

---

### Вопрос 8.4 🟡 Как работает событие (`event`) и почему это не просто публичное поле типа делегата?

**Ответ.** `event` — это языковая конструкция, которая оборачивает поле-делегат, предоставляя извне доступ **только** к `+=`/`-=` (add/remove), но не к прямому вызову или присвоению (`myEvent = null` снаружи класса недоступно):

```csharp
public class Publisher
{
    public event Action<string>? Message;   // event — не просто public Action<string>? Message;
    public void Raise(string text) => Message?.Invoke(text);
}
```

Если бы поле было объявлено просто как `public Action<string>? Message;`, любой внешний код мог бы полностью заменить список подписчиков (`publisher.Message = null;`), стерев чужие подписки — `event` предотвращает этот класс багов инкапсуляцией.

**Ресурсы.**
- [Events (C# Programming Guide)](https://learn.microsoft.com/dotnet/csharp/programming-guide/events/)

---

### Вопрос 8.5 🟡 Как безопасно передавать лямбду как обработчик события и затем отписаться от него?

**Ответ.** Чтобы отписаться (`-=`), нужна **та же ссылка** на делегат, что была передана при подписке. Инлайн-лямбда, объявленная прямо в `+=`, не может быть впоследствии передана в `-=` (так как каждая лямбда создаёт новый объект делегата), поэтому её нужно сохранить в переменную:

```csharp
EventHandler handler = (s, e) => Console.WriteLine("clicked");
button.Click += handler;
// ...
button.Click -= handler;   // корректно — та же ссылка на делегат
```

```csharp
button.Click += (s, e) => Console.WriteLine("clicked");
button.Click -= (s, e) => Console.WriteLine("clicked");  // НЕ отпишет — это другой объект делегата!
```

**Ресурсы.**
- [How to subscribe to and unsubscribe from events](https://learn.microsoft.com/dotnet/csharp/programming-guide/events/how-to-subscribe-to-and-unsubscribe-from-events)

---

### Вопрос 8.6 🟢 Что делают `Comparison<T>` и `Converter<TInput,TOutput>`, и как лямбда используется с ними?

**Ответ.** `Comparison<T>` — делегат `int Comparison<T>(T x, T y)`, используемый в `List<T>.Sort(Comparison<T>)`, `Array.Sort`. `Converter<TInput,TOutput>` — делегат `TOutput Converter<TInput,TOutput>(TInput input)`, используемый в `List<T>.ConvertAll`:

```csharp
list.Sort((a, b) => a.CompareTo(b));
List<string> strings = numbers.ConvertAll(n => n.ToString());
```

Оба — legacy-делегаты, предшествующие LINQ (появились ещё в .NET 2.0 вместе с generic-коллекциями); сегодня их функциональность в основном перекрывается LINQ (`OrderBy`, `Select`), но они остаются в BCL API и часто встречаются в вопросах "назовите как можно больше встроенных generic-делегатов".

**Ресурсы.**
- [Comparison<T> Delegate](https://learn.microsoft.com/dotnet/api/system.comparison-1)
- [Converter<TInput,TOutput> Delegate](https://learn.microsoft.com/dotnet/api/system.converter-2)

---

### Вопрос 8.7 🟡 Как реализовать паттерн "Strategy" через `Func`/`Action` вместо интерфейсов?

**Ответ.** Вместо иерархии классов, реализующих интерфейс с одним методом, можно передавать поведение напрямую как `Func`/`Action` — это уменьшает бойлерплейт для простых стратегий:

```csharp
public class Discount
{
    private readonly Func<decimal, decimal> _strategy;
    public Discount(Func<decimal, decimal> strategy) => _strategy = strategy;
    public decimal Apply(decimal price) => _strategy(price);
}

var tenPercentOff = new Discount(price => price * 0.9m);
var fiveDollarsOff = new Discount(price => Math.Max(0, price - 5));
```

Компромисс: интерфейсная реализация Strategy лучше документирует контракт (именованный тип, возможность нескольких методов, DI-регистрация по интерфейсу), тогда как `Func`-based подход компактнее для стратегий с единственной операцией и не требующих состояния помимо захваченных переменных.

**Ресурсы.**
- [Func<T,TResult> Delegate](https://learn.microsoft.com/dotnet/api/system.func-2)

---

## 9. Локальные функции vs лямбды

### Вопрос 9.1 🟡 В чём принципиальные отличия локальных функций от лямбда-выражений?

**Ответ.** Ключевые различия (C# 7+):

| Критерий | Лямбда | Локальная функция |
|---|---|---|
| Момент преобразования в делегат | Всегда, сразу при объявлении | Только если реально используется как делегат |
| Аллокация на heap | Обычно да (замыкание — класс) | Может отсутствовать (замыкание может быть `struct`, если функция не используется как делегат) |
| Рекурсия | Нужен трюк (см. 2.10) | Естественная — можно вызывать себя напрямую |
| `yield return` в теле | Не поддерживается | Поддерживается (может быть итератором) |
| Видимость до объявления | Нет — нужно объявить перед использованием | Да — можно вызвать до текстуального объявления в той же области видимости |
| `ref`/`out`/`in`/`this` (в struct) | Ограничения на захват | Может работать с ними напрямую, без обёртывания в делегат |

```csharp
IEnumerable<string> ToLowerAll(IEnumerable<string> input)
{
    return LowercaseIterator();  // вызов ДО объявления функции ниже — допустимо

    IEnumerable<string> LowercaseIterator()
    {
        foreach (var s in input)
            yield return s.ToLower();   // yield работает в локальной функции
    }
}
```

**Ресурсы.**
- [Local functions vs. lambda expressions](https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/local-functions#local-functions-vs-lambda-expressions)

---

### Вопрос 9.2 🟡 Почему локальные функции могут избежать heap-аллокации, а лямбды — почти никогда?

**Ответ.** Лямбда, будучи присвоенной делегату, **всегда** упаковывается в объект делегата (класс `Delegate`/`MulticastDelegate` — ссылочный тип), и если она захватывает переменные — дополнительно в объект-замыкание (тоже класс). Локальная функция преобразуется в делегат **только если её реально используют как делегат** (передают как аргумент типа `Func`/`Action`, сохраняют в переменную). Если локальная функция вызывается только напрямую по имени (как обычный метод), компилятор эмитит её как обычный приватный метод — без создания делегата вообще. А если она захватывает переменные, но не конвертируется в делегат, компилятор может реализовать замыкание как `struct`, передаваемый **по ссылке**, избегая heap-аллокации там, где для эквивалентной лямбды аллокация была бы неизбежна.

Рекомендация от Microsoft — помечать локальные функции модификатором `static`, когда они не должны захватывать состояние: это гарантирует отсутствие скрытого захвата и, соответственно, отсутствие аллокаций (правило стиля `IDE0062`).

**Ресурсы.**
- [Local functions — Heap allocations](https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/local-functions#local-functions-vs-lambda-expressions)
- [IDE0062: Make local function static](https://learn.microsoft.com/dotnet/fundamentals/code-analysis/style-rules/ide0062)

---

### Вопрос 9.3 🟢 Могут ли локальные функции быть `async`/итераторами одновременно с обычными методами?

**Ответ.** Да — локальная функция может быть помечена `async` (возвращая `Task`/`Task<T>`/`ValueTask<T>`) независимо от того, является ли объемлющий метод асинхронным, и может использовать `yield return` для реализации `IEnumerable<T>`/`IAsyncEnumerable<T>`. Частый паттерн — публичный метод сразу проверяет аргументы синхронно (fail-fast), а возвращает результат приватной локальной функции-итератора/асинхронной локальной функции, откладывающей фактическую работу:

```csharp
public IEnumerable<int> GetPositive(IEnumerable<int> source)
{
    ArgumentNullException.ThrowIfNull(source);   // проверка выполняется СРАЗУ, не откладываясь
    return Iterator();

    IEnumerable<int> Iterator()
    {
        foreach (var x in source)
            if (x > 0) yield return x;
    }
}
```

**Ресурсы.**
- [Local functions (C# Programming Guide)](https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/local-functions)

---

### Вопрос 9.4 🟡 Как definite assignment по-разному работает для локальных функций и лямбд при захвате переменных?

**Ответ.** См. также вопрос 3.10. Ключевое отличие: для локальных функций компилятор способен провести более глубокий статический анализ порядка вызова и присваивания, признавая переменную "точно присвоенной" внутри самой функции, если это гарантируется порядком выполнения кода в объемлющем методе. Для лямбд такой анализ не выполняется — переменная обязана быть присвоена **до** текстуального объявления лямбды.

**Ресурсы.**
- [Local functions — variable capture and definite assignment](https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/local-functions#local-functions-vs-lambda-expressions)

---

### Вопрос 9.5 🟢 Когда стоит предпочесть локальную функцию лямбде на практике?

**Ответ.** Общие рекомендации:
- Нужна **рекурсия** — локальная функция проще и без трюков с предварительным объявлением переменной.
- Функция используется только **внутри** объемлющего метода и вызывается напрямую (не передаётся как значение) — локальная функция избегает лишней аллокации и делает стек вызовов понятнее при отладке (локальные функции видны в call stack под своим именем, а не как `<>c__DisplayClass...`).
- Нужен `yield return`, но не хочется выносить логику в отдельный публичный метод.
- Нужна ранняя проверка аргументов при использовании итератора/асинхронного метода (fail-fast паттерн из вопроса 9.3).

Лямбда предпочтительнее, когда: функция **передаётся как значение** (в LINQ, как callback, как параметр `Func`/`Action`), либо когда она достаточно короткая, чтобы инлайн-запись повышала читаемость (`numbers.Where(n => n > 0)` явно лучше, чем вынос в отдельную локальную функцию).

**Ресурсы.**
- [Local functions vs. lambda expressions — summary](https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/local-functions#local-functions-vs-lambda-expressions)

---

### Вопрос 9.6 🟡 Может ли локальная функция быть обобщённой (generic) и иметь свои ограничения типов (constraints)?

**Ответ.** Да, локальные функции полностью поддерживают собственные generic-параметры и `where`-ограничения независимо от объемлющего метода:

```csharp
void Process<T>(IEnumerable<T> items) where T : class
{
    void LogAll<TItem>(IEnumerable<TItem> collection) where TItem : notnull
    {
        foreach (var item in collection)
            Console.WriteLine(item);
    }
    LogAll(items);
}
```

Лямбда-выражения такой возможности не имеют — они не могут объявлять собственные generic-параметры (лямбда всегда получает уже конкретные/выведенные типы из контекста присваивания). Это ещё одно структурное отличие, часто упускаемое на собеседовании.

**Ресурсы.**
- [Local functions (C# Programming Guide)](https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/local-functions)

---

## 10. Асинхронные лямбды и Task

### Вопрос 10.1 🟢 Как объявить асинхронную лямбду и когда это нужно?

**Ответ.** Добавлением модификатора `async` перед списком параметров лямбды, как и у обычного метода:

```csharp
button1.Click += async (sender, e) =>
{
    await ExampleMethodAsync();
    textBox1.Text += "\nControl returned to handler.";
};

Func<int, Task<int>> incrementAsync = async x => { await Task.Delay(10); return x + 1; };
```

Асинхронные лямбды нужны там, где API ожидает делегат, а логика внутри должна выполнить `await` — обработчики событий UI-фреймворков, callback-и в библиотеках, поддерживающих `Func<Task>`.

**Ресурсы.**
- [Async lambdas](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/lambda-expressions#async-lambdas)

---

### Вопрос 10.2 🔴 Почему `Parallel.ForEach` с асинхронной лямбдой — опасная ловушка, и как правильно распараллелить асинхронную работу?

**Ответ.** `Parallel.ForEach` принимает `Action<T>` как тело цикла. Если передать `async`-лямбду, она **всё ещё** соответствует сигнатуре `Action<T>` (компилятор допускает `async void`-совместимость с `Action`), но тогда делегат становится `async void` — вызывающий код не может отследить его завершение:

```csharp
// БАГ: завершается почти мгновенно, не дождавшись реальной работы
Parallel.ForEach(items, async item =>
{
    await Task.Delay(200);
});
```

`Parallel.ForEach` возвращается, как только каждый синхронный вызов дошёл до первого `await` (или до конца, если await'ов не было) — реальная асинхронная работа продолжается в фоне никем не отслеживаемая (fire-and-forget), а исключения из неё некому обработать.

**Правильные альтернативы:**
```csharp
// .NET 6+ : специализированный API для async-циклов
await Parallel.ForEachAsync(items, async (item, ct) => await ProcessAsync(item, ct));

// Или: спроецировать в задачи и дождаться всех
var tasks = items.Select(item => ProcessAsync(item));
await Task.WhenAll(tasks);
```

**Ресурсы.**
- [Async lambda pitfalls — Parallel.ForEach](https://learn.microsoft.com/dotnet/standard/asynchronous-programming-patterns/async-lambda-pitfalls#parallelforeach-with-async-lambdas)
- [Parallel.ForEachAsync](https://learn.microsoft.com/dotnet/api/system.threading.tasks.parallel.foreachasync)

---

### Вопрос 10.3 🔴 Чем опасен `Task.Factory.StartNew` с асинхронной лямбдой по сравнению с `Task.Run`?

**Ответ.** `Task.Run` **распознаёт** асинхронные делегаты — его перегрузки `Func<Task>`/`Func<Task<T>>` автоматически "разворачивают" (unwrap) внутреннюю задачу, и `await Task.Run(async () => await X())` корректно дожидается полного завершения `X()`.

`Task.Factory.StartNew`, напротив, **не** делает автоматический unwrap: если передать `Func<Task>`, `StartNew` вернёт `Task<Task>` — внешняя задача завершается сразу после синхронной части (до первого `await`), а внутренняя (настоящая) задача остаётся отдельным объектом:

```csharp
Task<Task> outer = Task.Factory.StartNew(async () => await Task.Delay(1000));
await outer;      // это завершится ПОЧТИ СРАЗУ — не дождавшись Delay!
// Правильно:
await outer.Unwrap();   // или сразу: await Task.Factory.StartNew(...).Unwrap();
```

Практический вывод: если не нужны специфичные опции `StartNew` (`TaskCreationOptions.LongRunning` и т.п.), почти всегда стоит предпочесть `Task.Run`. Если `StartNew` необходим — обязательно завершать вызов `.Unwrap()`.

**Ресурсы.**
- [Async lambda pitfalls — Task.Factory.StartNew](https://learn.microsoft.com/dotnet/standard/asynchronous-programming-patterns/async-lambda-pitfalls#taskfactorystartnew-with-async-lambdas)
- [TaskExtensions.Unwrap](https://learn.microsoft.com/dotnet/api/system.threading.tasks.taskextensions.unwrap)

---

### Вопрос 10.4 🔴 Что произойдёт, если асинхронную лямбду передать туда, где ожидается `Action`, а не `Func<Task>`?

**Ответ.** Компилятор допустит это (асинхронная лямбда без возвращаемого значения совместима как с `Func<Task>`, так и с `Action`), но во втором случае лямбда компилируется в `async void` метод. Основная проблема `async void`: исключения, брошенные внутри такой лямбды, **не попадают** в обычный механизм `try/catch` вызывающего кода — они "проскакивают" сразу в `SynchronizationContext`/поток, где могут привести к падению процесса, а не к перехватываемому исключению. Кроме того, вызывающий код не может ни дождаться завершения (`await`), ни получить результат:

```csharp
void MeasureTime(Action work)
{
    var sw = Stopwatch.StartNew();
    work();
    Console.WriteLine(sw.Elapsed);   // измерит только СИНХРОННУЮ часть асинхронной лямбды!
}

MeasureTime(async () => await Task.Delay(1000)); // напечатает ~0мс, а не ~1000мс
```

Правило: всегда проверять тип делегата целевого параметра перед передачей async-лямбды; если это `Action`/`Action<T>`, необходимо изменить сигнатуру API на `Func<Task>`/`Func<T,Task>`.

**Ресурсы.**
- [Async lambda pitfalls — summary table](https://learn.microsoft.com/dotnet/standard/asynchronous-programming-patterns/async-lambda-pitfalls#summary)

---

### Вопрос 10.5 🟡 Можно ли скомпилировать асинхронную лямбду в `Expression<Func<Task>>`?

**Ответ.** Нет — `await`-выражения и `async`-лямбды не могут быть представлены в виде expression tree (см. вопрос 7.3). Попытка присвоить асинхронную лямбду переменной типа `Expression<...>` приводит к ошибке компиляции. Это ограничение важно помнить при построении `IQueryable`-фильтров или динамических правил — вся логика внутри узла expression tree должна быть синхронной.

**Ресурсы.**
- [Expression Trees — Limitations (await/async)](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/#limitations)

---

### Вопрос 10.6 🟡 Как правильно захватывать переменные внешнего цикла в асинхронной лямбде, чтобы не наступить на ту же ловушку, что и с `for`?

**Ответ.** Проблема идентична синхронному случаю (вопрос 3.3), но чаще проявляется именно в асинхронном коде, так как между запуском задачи и её реальным выполнением проходит время, за которое цикл успевает "убежать" вперёд:

```csharp
var tasks = new List<Task>();
for (int i = 0; i < 3; i++)
{
    tasks.Add(Task.Run(async () =>
    {
        await Task.Delay(10);
        Console.WriteLine(i);     // почти наверняка все три задачи напечатают "3"
    }));
}
await Task.WhenAll(tasks);
```

Исправление — скопировать переменную цикла в локальную внутри тела `for` (как и в синхронном случае) либо использовать `foreach`, где с C# 5 каждая итерация уже создаёт новую переменную:

```csharp
for (int i = 0; i < 3; i++)
{
    int captured = i;
    tasks.Add(Task.Run(async () => { await Task.Delay(10); Console.WriteLine(captured); }));
}
```

**Ресурсы.**
- [Capture of outer variables and variable scope in lambda expressions](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/lambda-expressions#capture-of-outer-variables-and-variable-scope-in-lambda-expressions)

---

## 11. Pattern matching и switch expressions

### Вопрос 11.1 🟢 Чем `switch`-выражение отличается от `switch`-оператора?

**Ответ.** `switch`-**оператор** (statement) — управляющая конструкция, выполняющая блок кода в зависимости от совпадения с `case`; сам по себе не возвращает значения. `switch`-**выражение** — возвращает значение, синтаксис компактнее (нет `case`/`break`, ветви разделены запятыми, используется `=>`):

```csharp
// switch statement
string result;
switch (direction)
{
    case Direction.Up: result = "North"; break;
    case Direction.Down: result = "South"; break;
    default: result = "Unknown"; break;
}

// switch expression — то же самое компактнее
string result = direction switch
{
    Direction.Up => "North",
    Direction.Down => "South",
    _ => "Unknown",
};
```

`switch`-выражение — часть семейства "выбрать значение выражением" наряду с тернарным оператором `?:`; компилятор дополнительно предупреждает, если не все возможные входные значения обработаны (non-exhaustive switch).

**Ресурсы.**
- [`switch` expression](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/switch-expression)
- [Selection statements — select a value with an expression](https://learn.microsoft.com/dotnet/csharp/fundamentals/statements/selection#select-a-value-with-an-expression)

---

### Вопрос 11.2 🟢 Какие виды паттернов поддерживает pattern matching в C# (перечислите основные)?

**Ответ.**
1. **Declaration pattern** — `obj is Person p` (проверка runtime-типа + объявление переменной).
2. **Type pattern** — `obj is Person` (только проверка типа, без переменной).
3. **Constant pattern** — `x is 5`, `x is null`.
4. **Relational pattern** — `x is > 0 and < 100`.
5. **Logical patterns** — `and`, `or`, `not`: `x is not (float or double)`.
6. **Property pattern** — `person is { Age: > 18, Name.Length: > 0 }`.
7. **Positional pattern** — `point is (0, 0)` (использует `Deconstruct`).
8. **`var` pattern** — `x is var temp` (всегда совпадает, объявляет переменную).
9. **Discard pattern** — `x is _` (всегда совпадает, без переменной).
10. **List pattern** (C# 11) — `array is [1, 2, .., var last]`.

Все они применимы в `is`-выражении, `switch`-операторе и `switch`-выражении.

**Ресурсы.**
- [Pattern matching — the `is` and `switch` expressions](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/patterns)

---

### Вопрос 11.3 🟡 Что такое "exhaustiveness" (исчерпывающая обработка) в `switch`-выражении и как компилятор её проверяет?

**Ответ.** `switch`-выражение обязано вернуть значение для **любого** возможного входного значения; если ни одна ветвь не совпала во время выполнения, среда бросает `SwitchExpressionException` (в .NET Framework — `InvalidOperationException`). Компилятор пытается статически предупредить о такой ситуации, выдавая **предупреждение** (не ошибку) `CS8509`, если видит, что не все значения покрыты (например, не все значения `enum`, либо отсутствует discard-ветка для типа с открытым множеством значений):

```csharp
string Describe(Direction d) => d switch
{
    Direction.Up => "North",
    Direction.Down => "South",
    // CS8509: switch expression does not handle all possible values (например, Direction.Left)
};
```

Добавление discard-ветки `_ => "Unknown"` гарантирует исчерпывающую обработку для компилятора. Списковые паттерны (list patterns) — исключение: компилятор не предупреждает о неполноте для них, так как теоретически бесконечное множество длин списков сделало бы предупреждение малополезным.

**Ресурсы.**
- [Nonexhaustive switch expressions](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/switch-expression#nonexhaustive-switch-expressions)

---

### Вопрос 11.4 🟡 Что такое subsumption (поглощение одной ветви другой) и почему это ошибка компиляции в `switch`?

**Ответ.** Subsumption — ситуация, когда более ранняя (или более общая) ветвь `switch` заведомо перекрывает более позднюю, делая её недостижимой. Компилятор запрещает это как **ошибку** (не предупреждение) — в отличие от цепочки `if/else if`, где такая проблема осталась бы незамеченной:

```csharp
currentBalance += transaction switch
{
    (TransactionType.Deposit, var amount) => amount,
    _ => 0.0,                                              // перехватывает ВСЁ
    (TransactionType.Withdrawal, var amount) => -amount,   // ОШИБКА: недостижимо, subsumed
};
```

Это одно из практических преимуществ `switch`-выражений над цепочками `if/else if` — компилятор гарантированно ловит логическую ошибку порядка условий на этапе компиляции, а не в рантайме.

**Ресурсы.**
- [Tutorial: exhaustive matches with switch — subsumption example](https://learn.microsoft.com/dotnet/csharp/tour-of-csharp/tutorials/pattern-matching#exhaustive-matches-with-switch)

---

### Вопрос 11.5 🟡 Что такое case guard (`when`) в `switch`-выражении и зачем он нужен?

**Ответ.** Case guard — дополнительное булево условие после ключевого слова `when`, применяемое **после** успешного совпадения паттерна, когда самого паттерна недостаточно для выражения нужной логики:

```csharp
string Classify(Person p) => p switch
{
    { Age: var age } when age < 18 => "Minor",
    { Age: var age } when age >= 65 => "Senior",
    _ => "Adult",
};
```

Часто используется совместно с property-паттерном и вложенным `var`-паттерном, чтобы связать извлечённое значение с именем, доступным в условии `when`.

**Ресурсы.**
- [Case guards](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/switch-expression#case-guards)

---

### Вопрос 11.6 🟡 Как работает positional pattern и как он связан с деконструкцией (`Deconstruct`)?

**Ответ.** Positional pattern использует существующий или синтезированный метод `Deconstruct` типа, чтобы "разложить" объект на составляющие и сопоставить каждую с вложенным паттерном:

```csharp
record Point(int X, int Y);

string Describe(Point p) => p switch
{
    (0, 0) => "Origin",
    (var x, 0) => $"On X-axis at {x}",
    (0, var y) => $"On Y-axis at {y}",
    _ => "Elsewhere",
};
```

Для record-типов метод `Deconstruct` синтезируется автоматически на основе позиционных параметров основного конструктора. Для обычных классов/структур `Deconstruct` нужно объявить вручную (один или несколько перегруженных методов с `out`-параметрами), чтобы positional pattern заработал.

**Ресурсы.**
- [Positional pattern — Patterns](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/patterns#positional-pattern)

---

### Вопрос 11.7 🟡 Как список-паттерны (list patterns, C# 11) применяются к массивам и `List<T>`?

**Ответ.** Список-паттерны позволяют сопоставлять форму последовательности — фиксированные элементы в начале/конце и "срез" остатка через `..` (slice pattern):

```csharp
int[] numbers = { 1, 2, 3, 4, 5 };

string result = numbers switch
{
    [] => "empty",
    [var single] => $"one element: {single}",
    [var first, .., var last] => $"first={first}, last={last}",
};
```

Работает для массивов и любых типов с индексатором и свойством `Length`/`Count` (либо `Slice`-методом для срезов) — включая `List<T>`, `Span<T>`. Как отмечено в 11.3, компилятор не выдаёт предупреждение о неисчерпывающем покрытии для списковых паттернов.

**Ресурсы.**
- [List patterns](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/patterns#list-patterns)

---

### Вопрос 11.8 🟡 Как `switch`-выражение соотносится с closed hierarchy pattern (замкнутые иерархии, C# 15) для проверки полноты по типам?

**Ответ.** Начиная с C# 15, модификатор `closed` на классе/record фиксирует набор его прямых наследников на этапе компиляции. `switch`-выражение по такому базовому типу считается исчерпывающим, если обработаны **все прямые** наследники — без необходимости добавлять discard-ветку `_`:

```csharp
public closed record class PaymentMethod;
public record class Cash : PaymentMethod;
public record class Card(string Last4) : PaymentMethod;
public record class BankTransfer(string Iban) : PaymentMethod;

string Describe(PaymentMethod m) => m switch
{
    Cash => "cash",
    Card(var last4) => $"card ending {last4}",
    BankTransfer(var iban) => $"bank transfer to {iban}",
    // предупреждения нет — все прямые наследники обработаны
};
```

Нюанс с видимостью: если у `closed`-типа есть `internal`-наследник, невидимый из другой сборки, `switch` в этой сборке всё равно не будет считаться исчерпывающим (компилятор honest об этом предупреждает) — либо нужно добавить discard-ветку, либо обработать общий базовый тип.

**Диаграмма (замкнутая иерархия и exhaustiveness).**
```
        closed PaymentMethod
        ┌────────┼────────────┐
        ▼        ▼            ▼
      Cash     Card      BankTransfer
   (обработан)(обработан)  (обработан)

switch по PaymentMethod, где обработаны Cash/Card/BankTransfer → ИСЧЕРПЫВАЮЩИЙ, без "_"
```

**Ресурсы.**
- [Closed hierarchy patterns](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/patterns#closed-hierarchy-patterns)

---

### Вопрос 11.9 🟢 Как использовать `is`-выражение вместе с pattern matching для замены классического приведения типов?

**Ответ.**

```csharp
if (obj is Person { Age: >= 18 } adult)
{
    Console.WriteLine($"{adult.Name} is an adult");
}
```

Это заменяет старый паттерн "приведение + проверка на `null`" (`var p = obj as Person; if (p != null && p.Age >= 18) ...`) единой конструкцией, которая одновременно проверяет тип, проверяет вложенное условие через property pattern и объявляет типизированную переменную только в области видимости, где проверка прошла успешно (definite assignment гарантирует, что `adult` доступна только после успешного `is`).

**Ресурсы.**
- [Pattern matching overview](https://learn.microsoft.com/dotnet/csharp/fundamentals/functional/pattern-matching)

---

### Вопрос 11.10 🟡 Как правильно проверять на `null` с точки зрения паттернов, и почему `is null`/`is not null` предпочтительнее `== null`?

**Ответ.** `is null`/`is not null` — это **паттерн** (constant pattern), а не вызов оператора `==`. Разница критична, если тип переопределил `operator ==` с нестандартной логикой (например, вернул `true` для двух разных объектов) — `is null` всегда использует встроенную проверку на буквальное отсутствие ссылки, независимо от перегрузок оператора:

```csharp
if (value is null) { ... }        // всегда корректная проверка на null
if (value == null) { ... }        // может быть перехвачено перегруженным operator==
```

Это одна из рекомендаций в стилистических анализаторах (`IDE0041 — use is null check`) и общепринятая практика для кода, работающего с типами из сторонних библиотек, где неизвестно, как реализован `==`.

**Ресурсы.**
- [C# null operators — `is null`/`is not null`](https://learn.microsoft.com/dotnet/csharp/fundamentals/null-safety/null-operators)

---

## 12. Другие выражения C#

### Вопрос 12.1 🟢 Как работают `?.` (null-conditional) и `??` (null-coalescing), и как их комбинировать?

**Ответ.** `?.` выполняет обращение к члену/индексатору только если операнд слева не `null`; иначе всё выражение сразу возвращает `null` (short-circuit — правая часть цепочки не вычисляется вовсе):

```csharp
string? upper = input?.Trim()?.ToUpperInvariant();
```

`??` подставляет значение по умолчанию, если левый операнд — `null`. Комбинация — стандартный идиоматичный паттерн "безопасный доступ + дефолт":

```csharp
double sum = numbers?[index]?.Sum() ?? double.NaN;
```

Важная деталь: `?.` вычисляет операнд слева **не более одного раза**, гарантируя, что между проверкой на `null` и обращением к члену переменная не может "стать null" в многопоточном коде (thread-safe чтение для `Invoke` делегатов).

**Ресурсы.**
- [Member access operators — null-conditional](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/member-access-operators#null-conditional-operators--and-)
- [`??` and `??=` operators](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/null-coalescing-operator)

---

### Вопрос 12.2 🟢 Что делает `??=` и когда он избавляет от лишних вычислений?

**Ответ.** `variable ??= expr;` присваивает `expr` переменной только если она сейчас `null`; правая часть **не вычисляется вовсе**, если переменная уже не `null` — это не просто синтаксический сахар над `if`, а гарантия отсутствия побочных эффектов правой части при непустом значении:

```csharp
List<string>? cache = null;
cache ??= LoadData();   // LoadData() вызывается
cache ??= LoadData();   // LoadData() НЕ вызывается — cache уже не null
```

Классический паттерн ленивой инициализации без явного `if (x is null) x = ...;`.

**Ресурсы.**
- [Null-coalescing assignment `??=`](https://learn.microsoft.com/dotnet/csharp/fundamentals/null-safety/null-operators#null-coalescing-assignment--)

---

### Вопрос 12.3 🟡 Что нового в null-conditional assignment (C# 14) — `config?.Theme = "dark"`?

**Ответ.** До C# 14 `?.`/`?[]` можно было использовать только для **чтения**. Начиная с C# 14, их разрешили использовать как цель присваивания (в том числе составного, `+=`/`-=`, но не `++`/`--`):

```csharp
config?.Theme = "dark";               // присвоит, только если config не null
messages?[5] = "five";
values?[2] = GenerateNextIndex();     // GenerateNextIndex() вызовется, ТОЛЬКО если values не null
```

Важно: правая часть присваивания вычисляется **только тогда**, когда левая сторона гарантированно не `null` — это эквивалент `if (x is not null) x.Prop = expr;`, но компактнее. Такое выражение нельзя использовать как `ref`-переменную и нельзя передавать как `ref`/`out` аргумент.

**Ресурсы.**
- [Null-conditional assignment (C# 14)](https://learn.microsoft.com/dotnet/csharp/fundamentals/null-safety/null-operators#null-conditional-assignment-c-14)

---

### Вопрос 12.4 🟢 Что такое `throw`-выражение и где его можно использовать, в отличие от `throw`-оператора?

**Ответ.** С C# 7 `throw` можно использовать как **выражение** (а не только как оператор), что позволяет размещать его там, где синтаксически ожидается выражение: в тернарном операторе, `??`, expression-bodied членах:

```csharp
public string Name
{
    get => name;
    set => name = value ?? throw new ArgumentNullException(nameof(value));
}

string Description => _description ?? throw new InvalidOperationException("Not initialized");
```

До C# 7 такую проверку пришлось бы писать многословным блочным `if`, так как `throw`-оператор не мог стоять в позиции выражения.

**Ресурсы.**
- [The throw expression](https://learn.microsoft.com/dotnet/csharp/language-reference/statements/exception-handling-statements#the-throw-expression)

---

### Вопрос 12.5 🟡 Как работает `with`-выражение и какую копию оно создаёт — глубокую или поверхностную?

**Ответ.** `with`-выражение создаёт **поверхностную (shallow) копию** записи (`record`) или структуры с изменёнными указанными свойствами, оставляя исходный экземпляр неизменным (nondestructive mutation):

```csharp
var original = new Person("Grace", "Hopper");
var modified = original with { FirstName = "Margaret" };
Console.WriteLine(original == modified);  // False
```

Для record class компилятор синтезирует "clone"-метод и copy-конструктор; `with` вызывает clone, затем применяет изменения как обычные присваивания свойств. Так как копия поверхностная, для reference-типа свойства (например, `List<string>`) и оригинал, и копия будут указывать на **один и тот же** экземпляр списка — изменение элементов списка через одну ссылку будет видно через другую. Чтобы изменить это поведение, нужно написать собственный copy-конструктор.

**Диаграмма.**
```
original: Person { FirstName="Grace", LastName="Hopper", Tags=[list#1] }
                              │  with { FirstName = "Margaret" }
                              ▼
modified: Person { FirstName="Margaret", LastName="Hopper", Tags=[list#1] }
                                                                   ↑
                                              Tags указывает на ТОТ ЖЕ список — shallow copy!
```

**Ресурсы.**
- [The `with` expression](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/with-expression)
- [Records — Nondestructive mutation](https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/record#nondestructive-mutation)

---

### Вопрос 12.6 🟡 Почему вычисляемое свойство в record нужно объявлять как expression-bodied, а не через инициализатор, если используется `with`?

**Ответ.** См. также вопрос 4.4. Если вычисляемое свойство инициализируется один раз через `{ get; } = expr;`, `with`-выражение скопирует **уже вычисленное** значение поля, а не пересчитает его по новым данным — результат будет "залипшим" на старом состоянии:

```csharp
public record PointInit(int X, int Y)
{
    public double Distance { get; } = Math.Sqrt(X * X + Y * Y);  // вычислено ОДИН раз при создании
}
var p1 = new PointInit(3, 4);
var p2 = p1 with { Y = 8 };
Console.WriteLine(p2.Distance);  // всё ещё 5 — НЕПРАВИЛЬНО, должно быть ~8.54
```

Верно — сделать свойство expression-bodied (`=> Math.Sqrt(X * X + Y * Y);`), чтобы значение вычислялось заново при каждом обращении, включая обращение к уже скопированному через `with` экземпляру.

**Ресурсы.**
- [Records — computed properties and with expressions](https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/record#nondestructive-mutation)

---

### Вопрос 12.7 🟢 Что такое tuple-выражения и tuple-деконструкция?

**Ответ.** Value tuples (`(T1, T2, ...)`) позволяют группировать несколько значений без создания отдельного типа, и деконструировать их в отдельные переменные:

```csharp
(string first, string last) NameOf(Person p) => (p.First, p.Last);
var (first, last) = NameOf(somePerson);   // деконструкция в две переменные

// деконструкция произвольного типа через Deconstruct
var (x, y) = point;   // если Point объявляет public void Deconstruct(out int x, out int y)
```

Tuple-элементы могут иметь именованные метки (`first`, `last`), которые видны через IntelliSense и сохраняются компилятором в атрибутах, но не существуют в рантайме как настоящие имена полей — под капотом это `ValueTuple<T1,T2>` с полями `Item1`, `Item2`.

**Ресурсы.**
- [Value tuples — deconstruction](https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/value-tuples#deconstruction)

---

### Вопрос 12.8 🟡 Как работает target-typed `new` (`new()` без указания типа) и как это связано с выведением типов?

**Ответ.** Начиная с C# 9, `new()` может опустить имя типа, если тип однозначно выводится из контекста слева (объявление переменной, тип параметра, тип возврата):

```csharp
List<Order> orders = new();               // вместо new List<Order>();
Dictionary<string,int> counts = new() { ["a"] = 1 };
Point p = new(3, 4);                       // вызов конструктора с аргументами тоже поддерживается
```

Это отличается от `var` тем, что тип переменной **явно указан слева**, а `new()` лишь избегает повторения этого же имени типа справа — снижает избыточность в объявлениях с длинными generic-именами.

**Ресурсы.**
- [Target-typed new expressions](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-9#target-typed-new-expressions)

---

### Вопрос 12.9 🟡 В чём разница между `is`/`as` операторами и явным приведением типа `(T)obj` применительно к выражениям?

**Ответ.**
- `(T)obj` — явное приведение: бросает `InvalidCastException`, если приведение невозможно (для reference-типов) или `OverflowException`/потерю точности (для числовых типов, если не через `checked`/`unchecked` явно).
- `obj as T` — работает только для reference-типов и nullable value types; возвращает `null`, если приведение невозможно, вместо исключения.
- `obj is T` — просто проверка типа, возвращает `bool`, ничего не приводит (но, как выражение с типом-паттерном, может сразу же объявить переменную приведённого типа).

```csharp
object o = "hello";
int n = (int)o;          // InvalidCastException — o не int
string? s = o as string; // "hello" — успех
if (o is string str) { } // true, str = "hello"
```

Практический совет: `as` + проверка на `null` был стандартом до появления pattern matching; сегодня `is T variable` обычно предпочтительнее, так как объединяет проверку и приведение в одну атомарную конструкцию.

**Ресурсы.**
- [The `as` operator](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/type-testing-and-cast#the-as-operator)
- [The `is` operator](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/type-testing-and-cast#the-is-operator)

---

### Вопрос 12.10 🟡 Как приоритет и ассоциативность операторов влияют на интерпретацию сложных выражений с `??`, `?:` и лямбдами?

**Ответ.** Ключевые факты из таблицы приоритета операторов C#:
- `??` стоит **ниже** большинства операторов сравнения/логики, но **выше** `?:` (тернарного) — то есть `a ?? b ? c : d` разбирается как `(a ?? b) ? c : d`.
- `??` и `?:` — оба **правоассоциативны**: `a ?? b ?? c` = `a ?? (b ?? c)`; `x = y = z` = `x = (y = z)`.
- Лямбда-оператор `=>` тоже правоассоциативен, что важно для каррированных лямбд: `x => y => x + y` разбирается как `x => (y => x + y)` — функция, возвращающая функцию.

```csharp
Func<int, Func<int,int>> add = x => y => x + y;
Func<int,int> add5 = add(5);
Console.WriteLine(add5(3));  // 8
```

**Ресурсы.**
- [C# operators — precedence and associativity](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/#operator-precedence)

---

### Вопрос 12.11 🟢 Что такое conditional operator (`?:`) с точки зрения типа результата — почему иногда нужен явный каст?

**Ответ.** Тип результата тернарного оператора выводится как **общий тип** обеих ветвей. Если у типов ветвей нет неявной взаимной конвертации, компилятор выдаёт ошибку и требует явного приведения одной из ветвей:

```csharp
int x = 5;
var result = condition ? x : "text";        // ошибка CS0173 — нет общего типа между int и string
var result2 = condition ? (object)x : "text"; // OK — обе ветви приведены к object
```

Это тот же класс проблем, что и с "explicit return type" для лямбд (вопрос 1.4) — компилятору нужен единый статический тип результата выражения.

**Ресурсы.**
- [Conditional operator `?:`](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/conditional-operator)

---

### Вопрос 12.12 🟢 Как объектные и коллекционные инициализаторы соотносятся с "выражениями" в C#?

**Ответ.** Инициализаторы объектов (`new Person { Name = "A" }`) и коллекций (`new List<int> { 1, 2, 3 }`, а также collection expressions `[1, 2, 3]` из C# 12) — это тоже часть грамматики выражений: они порождают единое значение (новый объект/коллекцию) через комбинацию `new` и последовательности присваиваний/добавлений, скрытых за компактным синтаксисом:

```csharp
var person = new Person { Name = "Ada", Age = 36 };
int[] numbers = [1, 2, 3, 4, 5];              // collection expression, C# 12
List<int> list = [.. numbers, 6, 7];           // spread-оператор внутри collection expression
```

Collection expressions унифицируют синтаксис инициализации для массивов, `List<T>`, `Span<T>` и любых типов, поддерживающих коллекционные паттерны инициализации (`CollectionBuilderAttribute`), и являются одним из самых заметных нововведений последних версий C# в категории "выражений".

**Ресурсы.**
- [Collection expressions](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/collection-expressions)
- [Object and collection initializers](https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/object-and-collection-initializers)

---

## 13. Производительность и best practices

### Вопрос 13.1 🟡 Почему лямбда, объявленная внутри цикла и не захватывающая ничего, может быть кэширована компилятором, а захватывающая — нет?

**Ответ.** Если лямбда **не захватывает** ничего из окружения (ни локальных переменных, ни `this`), компилятор Roslyn кэширует единственный экземпляр делегата в статическом readonly поле сгенерированного вспомогательного класса и переиспользует его при каждом вызове метода — новый объект делегата создаётся **один раз за всё время жизни приложения**, а не на каждый вызов метода:

```csharp
IEnumerable<int> GetEvens(IEnumerable<int> src) => src.Where(x => x % 2 == 0); // делегат кэшируется компилятором
```

Если же лямбда захватывает переменную (даже параметр метода), она **не может** быть закэширована так же — на каждый вызов метода создаётся новый объект-замыкание (так как захваченное значение меняется от вызова к вызову):

```csharp
IEnumerable<int> GetGreaterThan(IEnumerable<int> src, int threshold)
    => src.Where(x => x > threshold);   // НОВАЯ аллокация замыкания на каждый вызов GetGreaterThan
```

Практический вывод для горячих путей: если возможно, выносите константное сравнение в статические лямбды, а параметризованное поведение по возможности передавайте через параметры метода, а не через захват.

**Ресурсы.**
- [Static lambdas](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/lambda-expressions#static-lambdas)

---

### Вопрос 13.2 🟡 Как избыточные аллокации от LINQ влияют на производительность в горячих путях, и какие альтернативы существуют?

**Ответ.** Каждый вызов `Where`/`Select` и т.п. создаёт новый объект-итератор (обычно на heap), плюс делегат для лямбды (если она захватывает состояние — ещё и объект-замыкание). В сценариях с высокой частотой вызова (обработка сообщений в очереди, hot path в игровом цикле, обработка большого числа HTTP-запросов) это создаёт заметное давление на GC. Альтернативы:
- Обычный `foreach` с ручной фильтрацией/агрегацией вместо цепочки LINQ-операторов — избегает промежуточных итераторов.
- `Span<T>`/`ReadOnlySpan<T>` вместе с ручными циклами для операций над массивами без аллокаций вовсе.
- Кэширование делегатов/лямбд в статических полях, если они не должны меняться между вызовами.
- Использование `for`-цикла по индексу вместо `foreach` там, где это даёт измеримую разницу (для массивов JIT неплохо оптимизирует оба варианта, но для некоторых коллекций `foreach` через интерфейс `IEnumerable<T>` даёт boxing итератора-структуры).

Важно: не стоит отказываться от LINQ повсеместно "на всякий случай" — в подавляющем большинстве кода (не hot path) читаемость важнее микрооптимизаций; профилирование должно предшествовать оптимизации.

**Ресурсы.**
- [Write efficient .NET code — Performance](https://learn.microsoft.com/dotnet/standard/garbage-collection/performance)

---

### Вопрос 13.3 🟡 Как выбор между `Func<T,bool>` (делегат) и `Expression<Func<T,bool>>` (дерево) влияет на производительность в конкретном случае?

**Ответ.** Компиляция делегата через `.Compile()` — дорогая одноразовая операция, но сам вызов скомпилированного делегата быстр (обычная JIT-скомпилированная функция). Если предикат используется **много раз** над **in-memory** коллекцией (LINQ to Objects), нет причин строить `Expression<Func<T,bool>>` — сразу используйте `Func<T,bool>`, чтобы избежать накладных расходов на построение и компиляцию дерева. `Expression<Func<T,bool>>` оправдан только тогда, когда дерево реально нужно **как данные** — для трансляции в SQL (`IQueryable`), анализа, сериализации или динамической композиции условий.

```csharp
// LINQ to Objects — не нужно Expression, только лишние накладные расходы
list.Where((Func<int,bool>)(x => x > 0));

// LINQ to Entities — обязательно Expression, чтобы EF Core перевёл в SQL
dbSet.Where((Expression<Func<Order,bool>>)(o => o.Total > 0));
```

**Ресурсы.**
- [Lambdas with the standard query operators](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/lambda-expressions#lambdas-with-the-standard-query-operators)

---

### Вопрос 13.4 🟡 Почему передача метода как method group иногда эффективнее лямбды и наоборот?

**Ответ.** Для **статического** метода без захвата method group conversion (вопрос 2.5) не требует создания объекта-замыкания — просто указатель на метод оборачивается в делегат, что почти эквивалентно по цене простой лямбде без захвата (обе кэшируются компилятором как одноразовые статические делегаты, см. 13.1). Для **метода экземпляра**, переданного как method group (`instance.Method`), делегат захватывает ссылку на `instance` — это концептуально то же самое, что лямбда `x => instance.Method(x)`, и кэшировать такой делегат между разными экземплярами `instance` нельзя (каждый вызов с новым `instance` создаёт новый делегат). На практике разница между method group и эквивалентной лямбдой по производительности пренебрежимо мала — выбор обычно определяется читаемостью, а не производительностью.

**Ресурсы.**
- [Method group conversions](https://learn.microsoft.com/dotnet/csharp/language-reference/language-specification/conversions)

---

### Вопрос 13.5 🟢 Как избежать "closure over loop variable" в современном коде, помимо ручного копирования переменной?

**Ответ.** Помимо паттерна "скопировать в новую локальную переменную внутри тела цикла" (вопрос 3.3), современные альтернативы:
- Использовать `foreach` вместо `for`, когда это возможно — начиная с C# 5, переменная итерации `foreach` per-iteration, что автоматически устраняет проблему.
- Использовать LINQ (`Select`, `Where`), где параметр лямбды — это уже "своя" переменная на каждый вызов, а не общая переменная внешнего цикла.
- Включить анализатор Roslyn — многие статические анализаторы (включая встроенные предупреждения компилятора в некоторых версиях, и обязательно линтеры типа ReSharper/Rider) явно подсвечивают захват изменяемой переменной цикла внутри замыкания как code smell.

**Ресурсы.**
- [Capture of outer variables and variable scope](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/lambda-expressions#capture-of-outer-variables-and-variable-scope-in-lambda-expressions)

---

### Вопрос 13.6 🟡 Как аллокации замыканий проявляются при профилировании (что искать в дампах памяти/бенчмарках)?

**Ответ.** В дампах памяти (WinDbg/dotnet-dump, PerfView) замыкания видны как экземпляры сгенерированных компилятором классов с именами вида `<>c__DisplayClass0_0`, а кэшированные делегаты без захвата — как экземпляры `<>c` (singleton-класс для статических/некапчурящих лямбд метода). В BenchmarkDotNet — колонка `Allocated` резко возрастает при переходе от кода без замыканий к коду, создающему замыкание на каждый вызов бенчмарка. Частый паттерн диагностики — сравнить два бенчмарка: один с лямбдой, захватывающей параметр метода, другой — с эквивалентной локальной функцией/явным `static`-делегатом, и сопоставить `Gen0`/`Allocated`.

**Ресурсы.**
- [BenchmarkDotNet — MemoryDiagnoser](https://benchmarkdotnet.org/articles/configs/diagnosers.html)

---

### Вопрос 13.7 🟢 Какие практические правила стиля рекомендуются для лямбд в командном код-стайле?

**Ответ.** Часто применяемые на практике рекомендации:
- Использовать `static` модификатор для лямбд, которые не должны захватывать состояние (документирует намерение + защищает от случайного захвата, вопрос 3.6).
- Явно именовать параметры содержательно, а не только `x`, `y`, когда лямбда достаточно длинная или вложенная (`order => order.Total`, а не `o => o.Total`, в сложных цепочках).
- Ограничивать длину statement lambda — если тело растёт до нескольких экранов, вынести в отдельный именованный метод/локальную функцию.
- Избегать side-effects внутри `Select`/`Where` (проекция должна быть чистой функцией; побочные эффекты внутри LINQ-запросов усложняют рассуждение о повторных/множественных enumeration, вопрос 5.9).
- Настроить `.editorconfig` с явными правилами `csharp_style_expression_bodied_*`, чтобы избежать споров в code review о стиле.

**Ресурсы.**
- [.NET code style rules overview](https://learn.microsoft.com/dotnet/fundamentals/code-analysis/style-rules/)

---

### Вопрос 13.8 🟡 Как `record`/`with`-выражения могут неожиданно провоцировать лишние аллокации по сравнению с мутируемыми классами?

**Ответ.** Каждый `with`-вызов создаёт **новый** объект (для `record class`) — если такой код выполняется в цикле с частыми "изменениями" одного и того же логического объекта, это может генерировать значительно больше аллокаций, чем изменение поля обычного mutable-класса на месте:

```csharp
// N аллокаций новых объектов вместо изменения одного
Person current = initial;
foreach (var update in updates)
    current = current with { Age = update.NewAge };
```

Для `record struct` цена другая — копирование происходит по значению (без heap-аллокации на каждый `with`, если сама структура не boxed), что делает `record struct` предпочтительным выбором для небольших часто мутируемых через `with` значений в горячих путях. Выбор между `record class` и `record struct` (вопрос из общего C#, но тесно связан с `with`-выражениями) должен учитывать частоту таких операций.

**Ресурсы.**
- [C# record types — record class vs record struct](https://learn.microsoft.com/dotnet/csharp/fundamentals/types/records#nondestructive-mutation-with-with-expressions)

---

## 14. Advanced/Guru темы

### Вопрос 14.1 🔴 Как работает вариантность (`in`/`out`) для делегатов `Func`/`Action`, и почему `Func<Derived,Base>` можно присвоить `Func<Base,Derived>`-совместимой переменной, а не наоборот?

**Ответ.** `Func<in T, out TResult>` объявлен с контравариантным входным параметром (`in T`) и ковариантным результатом (`out TResult`). Это означает:
- **Ковариантность результата** — метод, возвращающий более производный тип, можно использовать там, где ожидается делегат с менее производным типом результата (`Employee Foo(...)` можно присвоить `Func<..., Person>`, если `Employee : Person`).
- **Контравариантность параметра** — метод, принимающий менее производный (более общий) тип параметра, можно использовать там, где ожидается делегат с более производным типом параметра (`void Foo(Person p)` можно присвоить `Action<Employee>`, если `Employee : Person`).

```csharp
class Person { }
class Employee : Person { }

Func<Person, Employee> f1 = p => new Employee();

// Ковариантный результат: Employee более производный, чем Base — можно "сузить" ожидание результата до Base
Func<Person, Person> f2 = f1;

// Контравариантный параметр: можно "расширить" тип параметра, если исходный метод принимал более общий тип
Func<Employee, Employee> f3 = f1;

Func<Employee, Person> f4 = f1;  // оба эффекта сразу
```

Мнемоника: **"input contravariant, output covariant"** — на входе можно принимать *более общий* тип, чем требуется (безопасно передать `Employee`, если метод и так принимает любой `Person`), а на выходе можно возвращать *более специфичный* тип, чем ожидается (любой `Employee` — это тоже `Person`).

**Диаграмма (иерархия и направление стрелок вариантности).**
```
                     Person  (базовый)
                        ▲
                        │  (наследование)
                     Employee (производный)

Func<in T, out TResult>:
   T (параметр)      — CONTRAVARIANT (in): разрешено "расширять" вниз по стрелке присваивания
   TResult (результат) — COVARIANT (out): разрешено "сужать" вниз по стрелке присваивания

   Func<Person, Employee>  ──assignable to──▶  Func<Employee, Person>
        (шире вход)                                (уже вход, шире выход — оба безопасны)
```

**Ресурсы.**
- [Variance in Delegates (C#)](https://learn.microsoft.com/dotnet/csharp/programming-guide/concepts/covariance-contravariance/variance-in-delegates)
- [Using Variance for Func and Action Generic Delegates](https://learn.microsoft.com/dotnet/csharp/programming-guide/concepts/covariance-contravariance/using-variance-for-func-and-action-generic-delegates)

---

### Вопрос 14.2 🔴 Почему нельзя объединять (`Combine`/`+=`) вариантно совместимые, но не идентичные делегаты?

**Ответ.** `Delegate.Combine` (и оператор `+`) требует, чтобы оба делегата были **одного и того же точного типа времени выполнения** — вариантная совместимость на этапе компиляции (позволяющая присвоить один вариантный делегат переменной другого вариантного типа) не распространяется на комбинирование через multicast:

```csharp
Action<object> actObj = x => Console.WriteLine("object: {0}", x);
Action<string> actStr = x => Console.WriteLine("string: {0}", x);

// Все следующие строки бросят исключение в рантайме:
// Action<string> actCombine = actStr + actObj;
// actStr += actObj;
// Delegate.Combine(actStr, actObj);
```

Причина — `Combine` реализован на уровне `MulticastDelegate` без знания о вариантности конкретного generic-делегата; он сравнивает типы напрямую. Это тонкость, которую часто упускают даже опытные разработчики, ожидая, что раз присваивание работает, то и объединение сработает так же.

**Ресурсы.**
- [Variance in Delegates — Combining Variant Generic Delegates](https://learn.microsoft.com/dotnet/csharp/programming-guide/concepts/covariance-contravariance/variance-in-delegates#variance-in-generic-type-parameters)

---

### Вопрос 14.3 🔴 Почему вариантность делегатов/generic-интерфейсов не поддерживается для value-типов?

**Ответ.** Вариантность в CLR реализована на уровне ссылок — она работает, полагаясь на то, что объект одного типа можно неявно трактовать как объект другого (совместимого по CLR-иерархии) типа **без изменения представления в памяти**. Value-типы (`int`, `struct`) имеют принципиально другое представление в памяти (нет единого "заголовка объекта" с указателем на таблицу методов, совместимого между разными value-типами), поэтому CLR не поддерживает implicit reference conversion для generic-параметров, инстанцированных value-типами:

```csharp
public delegate T SampleGenericDelegate<out T>();
SampleGenericDelegate<int> dInt = () => 5;
// SampleGenericDelegate<object> dObj = dInt;  // ОШИБКА — int является value-типом, вариантность не работает
```

Это стандартный "gotcha"-вопрос: кандидат должен объяснить, что ковариантность/контравариантность в C#/CLR — это исключительно про reference-типы generic-аргументов, независимо от того, объявлен ли сам параметр как `in`/`out`.

**Ресурсы.**
- [Variance in Delegates — Value and Reference Types](https://learn.microsoft.com/dotnet/csharp/programming-guide/concepts/covariance-contravariance/variance-in-delegates#variance-in-generic-type-parameters-for-value-and-reference-types)

---

### Вопрос 14.4 🔴 Как реализовать эффективный доступ к свойству объекта через reflection + Expression Trees вместо `PropertyInfo.GetValue`?

**Ответ.** Прямой вызов `PropertyInfo.GetValue(obj)` относительно медленный (полный путь через reflection на каждый вызов, включая проверки безопасности и boxing для value-типов). Быстрая альтернатива — построить и **закэшировать** скомпилированный geter через `Expression`:

```csharp
static Func<T, object?> BuildGetter<T>(PropertyInfo property)
{
    var target = Expression.Parameter(typeof(T), "target");
    var propertyAccess = Expression.Property(target, property);
    var castToObject = Expression.Convert(propertyAccess, typeof(object));
    var lambda = Expression.Lambda<Func<T, object?>>(castToObject, target);
    return lambda.Compile();   // компилируем ОДИН раз и кэшируем результат
}

// использование:
var getter = BuildGetter<Person>(typeof(Person).GetProperty(nameof(Person.Name))!);
object? name = getter(somePerson);   // дальше — обычный вызов делегата, быстро
```

После первичной компиляции такой getter по скорости сопоставим с обычным вызовом свойства напрямую (нет reflection-overhead на каждый вызов) — этот паттерн используют сериализаторы, мапперы (AutoMapper) и ORM.

**Ресурсы.**
- [Build expression trees](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-building)

---

### Вопрос 14.5 🔴 Что такое System.Linq.Dynamic.Core и как оно относится к Expression Trees?

**Ответ.** Это сторонняя (но широко используемая в индустрии) библиотека, позволяющая строить LINQ-запросы **из строк** во время выполнения (например, "Age > 18 AND Name.StartsWith(\"A\")", введённая пользователем в UI-фильтр), парся такую строку в expression tree и подставляя её как `Expression<Func<T,bool>>` в `IQueryable<T>.Where(...)`:

```csharp
IQueryable<Person> query = dbSet.Where("Age > 18 AND Name.StartsWith(@0)", "A");
```

Внутри библиотека реализует собственный мини-парсер выражений (аналог того, что компилятор C# делает статически), который на лету строит те же самые классы `System.Linq.Expressions.Expression`, что и обычный компилятор C#, — просто на основе текстовой строки, а не C#-синтаксиса. Это яркая демонстрация того, что Expression Trees — универсальный публичный API, не привязанный жёстко к компилятору C#.

**Ресурсы.**
- [System.Linq.Expressions namespace](https://learn.microsoft.com/dotnet/api/system.linq.expressions)
- [Dynamic LINQ — сторонний проект (NuGet: System.Linq.Dynamic.Core)]

---

### Вопрос 14.6 🔴 Что происходит "под капотом" при компиляции лямбды в делегат: сколько классов и методов генерирует компилятор в разных сценариях?

**Ответ.** Roslyn применяет разные стратегии генерации в зависимости от того, что лямбда захватывает:

1. **Ничего не захватывает (static-совместимая лямбда)** → метод генерируется как `private static` метод сгенерированного класса `<>c` (singleton display class на весь тип), делегат кэшируется в static readonly поле — **одна аллокация делегата за всё время жизни процесса**.
2. **Захватывает только `this`** → метод становится обычным приватным методом экземпляра исходного класса — делегат создаётся на каждый вызов метода-владельца (так как нужна ссылка на конкретный `this`), но **без** отдельного класса-замыкания.
3. **Захватывает локальные переменные** → генерируется отдельный класс `<>c__DisplayClassN_M` с полями под каждую захваченную переменную; метод лямбды становится методом экземпляра этого класса; на каждый вызов метода-владельца создаётся новый экземпляр этого класса (если переменные, захватываемые разными лямбдами, объявлены в разных вложенных областях — может быть несколько уровней таких классов, ссылающихся друг на друга).
4. **Захватывает `this` и локальные переменные одновременно** → `this` тоже становится полем display-класса, чтобы унифицировать доступ.

Проверить это на практике проще всего через [SharpLab.io](https://sharplab.io) в режиме "C# → C#" (decompile), который явно покажет сгенерированные классы `<>c` / `<>c__DisplayClassN_M`.

**Ресурсы.**
- [Local functions — closure implementation details](https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/local-functions#local-functions-vs-lambda-expressions)
- Eric Lippert, "Closures examined" (серия статей в блоге о деталях реализации Roslyn)

---

### Вопрос 14.7 🔴 Как лямбды взаимодействуют с ковариантностью возвращаемых типов в C# 9+ (`covariant return types`) на уровне переопределения методов?

**Ответ.** Начиная с C# 9, переопределённый метод может сузить (covariant) тип возврата по сравнению с базовым — это отдельная фича от вариантности делегатов (вопрос 14.1), но пересекается с ней, когда переопределённый метод затем используется как источник method group conversion в делегат:

```csharp
class Animal { public virtual Animal Reproduce() => new Animal(); }
class Dog : Animal { public override Dog Reproduce() => new Dog(); }  // covariant return, C# 9+

Func<Animal> f = new Dog().Reproduce;   // method group conversion работает благодаря variance Func<out TResult>
```

Без ковариантных возвращаемых типов пришлось бы явно приводить (`() => (Animal)new Dog().Reproduce()`), либо переопределённый метод должен был бы возвращать ровно `Animal`. Связка "covariant return types в переопределении" + "вариантность делегатов" — комбинированный advanced-вопрос, который проверяет понимание обеих фич одновременно.

**Ресурсы.**
- [Covariant return types](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-9#covariant-return-types)

---

### Вопрос 14.8 🟡 Как работает `nameof` в контексте лямбд и expression trees, и почему это полезно для рефакторинг-безопасного кода?

**Ответ.** `nameof(x)` — компайл-тайм оператор, возвращающий строковое имя идентификатора; он не является выражением, из которого строится expression tree само по себе, но часто применяется **вместе** с expression trees в библиотеках вроде FluentValidation, где имя свойства извлекается из лямбды через анализ `Expression<Func<T,TProperty>>`, а не через `nameof` напрямую:

```csharp
// Подход 1: nameof — просто строка имени параметра/члена, известная на этапе компиляции
throw new ArgumentNullException(nameof(value));

// Подход 2: извлечение имени свойства ИЗ expression tree (используется в валидаторах/ORM)
static string GetMemberName<T,TProp>(Expression<Func<T,TProp>> expr)
{
    var member = (MemberExpression)expr.Body;
    return member.Member.Name;
}
string propName = GetMemberName<Person,string>(p => p.Email);   // "Email"
```

Оба подхода рефакторинг-безопасны (переименование через IDE обновит и `nameof`, и лямбду), в отличие от "magic string" `"Email"`, вписанной вручную.

**Ресурсы.**
- [nameof expression](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/nameof)

---

### Вопрос 14.9 🔴 Как ограничения на конструкции внутри expression trees (раздел 7.3) исторически менялись между версиями C#, и почему это осознанное архитектурное решение?

**Ответ.** Начиная с C# 6, в язык добавлялось много новых конструкций (null-conditional `?.`, интерполированные строки, `nameof`), но `System.Linq.Expressions` **не получал новых типов узлов** для каждой такой фичи. Причина — `Expression`/`ExpressionType` — публичный, широко используемый API (Entity Framework и десятки других библиотек пишут собственные `ExpressionVisitor`, ожидающие конечный, стабильный набор `NodeType`). Добавление нового типа узла было бы **breaking change** для всех таких визитёров — они не знали бы, как обработать незнакомый узел, и падали бы или работали бы некорректно.

Решение архитекторов языка — там, где возможно, **разворачивать** новую фичу в эквивалентную комбинацию **старых, уже существующих** узлов при построении expression tree (например, `?.` до определённой версии либо был запрещён в expression tree, либо трансформировался в `Conditional`+`null`-проверку на старых примитивах), а там, где это невозможно (`await`, `async`, локальные функции, `dynamic`), — просто **запрещать** использование этой фичи внутри expression lambda на этапе компиляции.

**Ресурсы.**
- [Expression Trees — Limitations](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/#limitations)

---

### Вопрос 14.10 🔴 Как реализовать интерпретатор (evaluator) для собственного мини-DSL на основе Expression Trees, обходя дерево рекурсивно?

**Ответ.** Общий алгоритм — рекурсивная функция, которая для каждого `NodeType` знает, как вычислить (или, для трансляции, — сгенерировать целевой код) для этого узла, рекурсивно обрабатывая дочерние узлы:

```csharp
static object? Evaluate(Expression expr) => expr switch
{
    ConstantExpression c => c.Value,
    BinaryExpression { NodeType: ExpressionType.Add } b =>
        (int)Evaluate(b.Left)! + (int)Evaluate(b.Right)!,
    BinaryExpression { NodeType: ExpressionType.Multiply } b =>
        (int)Evaluate(b.Left)! * (int)Evaluate(b.Right)!,
    ConditionalExpression cond =>
        (bool)Evaluate(cond.Test)! ? Evaluate(cond.IfTrue) : Evaluate(cond.IfFalse),
    MethodCallExpression call =>
        call.Method.Invoke(
            call.Object is null ? null : Evaluate(call.Object),
            call.Arguments.Select(Evaluate).ToArray()),
    _ => throw new NotSupportedException($"Unsupported node: {expr.NodeType}")
};
```

Это по сути то, что делает `Compile()` внутри CLR (только компилируя в IL, а не интерпретируя рекурсивно), и то, что делают ORM-провайдеры (только вместо вычисления значения — генерируя фрагмент SQL для каждого узла). Понимание этого паттерна — хороший показатель "guru"-уровня владения темой на собеседовании, так как объединяет знания о `NodeType`, `ExpressionVisitor` и практическое применение в реальных библиотеках.

**Ресурсы.**
- [Interpret expressions](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-interpreting)
- [Translating expression trees](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-translating)

---

### Вопрос 14.11 🔴 Почему `Expression<Func<T,bool>>`, полученный из разных, но текстуально идентичных лямбд, не равен через `==`, и как EF Core кэширует такие запросы несмотря на это?

**Ответ.** `Expression`-объекты — обычные ссылочные типы без переопределённого `Equals`/`==` по значению; сравнение двух отдельно построенных (даже текстуально идентичных) деревьев через `==` или `.Equals()` сравнивает **ссылки**, а не структуру, и всегда будет `false` для разных построений:

```csharp
Expression<Func<int,bool>> e1 = x => x > 5;
Expression<Func<int,bool>> e2 = x => x > 5;
Console.WriteLine(e1 == e2);          // False — разные объекты
Console.WriteLine(e1.Equals(e2));     // False — ссылочное сравнение по умолчанию
```

EF Core решает задачу кэширования скомпилированных запросов не через `Equals` дерева целиком, а через собственный механизм **сравнения по структуре** (`ExpressionEqualityComparer`) плюс кэш скомпилированных query-планов, ключом к которому служит нормализованное представление дерева запроса (с "дырками" под параметризуемые константы) — это отдельная сложная подсистема (`QueryCompilationContext`), которая выходит за рамки публичного API `System.Linq.Expressions`, но само явление ("одинаковый с виду код — разные объекты expression tree") — стандартный вопрос уровня guru.

**Ресурсы.**
- [ExpressionEqualityComparer Class](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expressionequalitycomparer)

---

### Вопрос 14.12 🟡 Как лямбды и expression trees взаимодействуют с nullable reference types (NRT) — есть ли особенности аннотирования?

**Ответ.** Лямбды полностью поддерживают nullable-аннотации параметров и возврата так же, как обычные методы (`Func<string?, int?>`), и анализатор потока nullable-состояния (null-state) внутри тела лямбды работает так же, как в обычном коде — включая распространение состояния "проверено на null" внутрь замыкания в рамках одного вызова. Особенность: анализатор **не может** гарантировать, что состояние захваченной переменной, проверенной на non-null **перед** объявлением лямбды, останется non-null **к моменту фактического вызова** делегата (который может произойти значительно позже, в другом потоке) — поэтому компилятор консервативно иногда всё равно требует повторной проверки внутри тела лямбды, если переменная объявлена как nullable.

**Ресурсы.**
- [Nullable reference types](https://learn.microsoft.com/dotnet/csharp/nullable-references)

---


## Итоговый список общих ресурсов

- [Lambda expressions and anonymous functions — C# language reference](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/lambda-expressions)
- [Lambda expressions, delegates, and events — Fundamentals](https://learn.microsoft.com/dotnet/csharp/fundamentals/types/delegates-lambdas)
- [Expression Trees — overview and building/executing/translating/interpreting](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/)
- [System.Linq.Expressions namespace reference](https://learn.microsoft.com/dotnet/api/system.linq.expressions)
- [Pattern matching — is/switch expressions and patterns](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/patterns)
- [LINQ queries — Fundamentals](https://learn.microsoft.com/dotnet/csharp/fundamentals/statements/linq)
- [Introduction to LINQ Queries — deferred execution classification](https://learn.microsoft.com/dotnet/csharp/linq/get-started/introduction-to-linq-queries)
- [Covariance and Contravariance in C#](https://learn.microsoft.com/dotnet/csharp/programming-guide/concepts/covariance-contravariance/)
- [Local functions (C# Programming Guide)](https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/local-functions)
- [Async lambda pitfalls](https://learn.microsoft.com/dotnet/standard/asynchronous-programming-patterns/async-lambda-pitfalls)
- [C# null operators (?. ?? ??=)](https://learn.microsoft.com/dotnet/csharp/fundamentals/null-safety/null-operators)
- [The `with` expression / Records](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/with-expression)
- [Errors and warnings when using lambda expressions](https://learn.microsoft.com/dotnet/csharp/language-reference/compiler-messages/lambda-expression-errors)
- [C# language specification — Expressions (§12)](https://learn.microsoft.com/dotnet/csharp/language-reference/language-specification/expressions)
- [C# version history (когда какая фича появилась)](https://learn.microsoft.com/dotnet/csharp/whats-new/csharp-version-history)
- [.editorconfig code style rules for expression-bodied members](https://learn.microsoft.com/dotnet/fundamentals/code-analysis/style-rules/)
- Книга: Jon Skeet, *"C# in Depth"* (4th/5th edition) — главы о лямбдах, замыканиях, LINQ и expression trees
- Блог: Eric Lippert, серия *"Closures examined"* и материалы о деталях реализации C#-компилятора
- Инструмент: [SharpLab.io](https://sharplab.io) — просмотр IL/decompiled C# для любой лямбды, чтобы увидеть сгенерированные классы замыканий

---

*Документ подготовлен как база для подготовки к техническим собеседованиям уровня middle → senior → staff/principal по C#/.NET, с акцентом на глубокое понимание лямбд, делегатов, LINQ и expression trees, а не только на синтаксис.*
