# C# Expression Trees — 200+ вопросов для собеседований (Intermediate → Advanced/Guru)

> Подборка вопросов и детальных ответов по `System.Linq.Expressions` — от базового понимания `Expression<TDelegate>` до написания собственного LINQ-провайдера. Вопросы сгруппированы по темам, каждый снабжён ответом, примерами кода, ссылками на официальную документацию и (там, где это упрощает понимание) схемой.
>
> Уровень сложности отмечен тегами: **[I]** — Intermediate, **[A]** — Advanced, **[G]** — Guru.

---

## Оглавление

1. [Основы и концепция Expression Trees](#группа-1-основы-и-концепция-expression-trees) (10)
2. [`Expression<TDelegate>` vs `Func`/`Action`, лямбды](#группа-2-expressiontdelegate-vs-funcaction-лямбды) (8)
3. [Анатомия узлов дерева (типы Expression)](#группа-3-анатомия-узлов-дерева-типы-expression) (15)
4. [Построение деревьев вручную (Expression API)](#группа-4-построение-деревьев-вручную-expression-api) (12)
5. [ExpressionVisitor: обход и трансформация](#группа-5-expressionvisitor-обход-и-трансформация) (12)
6. [Компиляция: `Compile()` изнутри](#группа-6-компиляция-compile-изнутри) (10)
7. [LINQ to Objects vs LINQ to Queryable](#группа-7-linq-to-objects-vs-linq-to-queryable) (10)
8. [EF Core и провайдеры: трансляция в SQL](#группа-8-ef-core-и-провайдеры-трансляция-в-sql) (12)
9. [Динамические предикаты и `PredicateBuilder`](#группа-9-динамические-предикаты-и-predicatebuilder) (10)
10. [Dynamic LINQ (`System.Linq.Dynamic.Core`)](#группа-10-dynamic-linq-systemlinqdynamiccore) (6)
11. [Ограничения expression trees и эволюция C#](#группа-11-ограничения-expression-trees-и-эволюция-c) (10)
12. [Замыкания (closures) в expression trees](#группа-12-замыкания-closures-в-expression-trees) (8)
13. [Структурное сравнение деревьев](#группа-13-структурное-сравнение-деревьев) (6)
14. [Производительность](#группа-14-производительность) (10)
15. [Практические сценарии: AutoMapper, Moq, Specification](#группа-15-практические-сценарии-automapper-moq-specification) (10)
16. [Отладка expression trees](#группа-16-отладка-expression-trees) (6)
17. [Сериализация expression trees](#группа-17-сериализация-expression-trees) (6)
18. [Expression Trees vs Reflection.Emit vs Source Generators](#группа-18-expression-trees-vs-reflectionemit-vs-source-generators) (8)
19. [Nullable, boxing и `Convert`](#группа-19-nullable-boxing-и-convert) (6)
20. [Ошибки построения и компиляции](#группа-20-ошибки-построения-и-компиляции) (6)
21. [Guru: пишем мини LINQ-провайдер](#группа-21-guru-пишем-мини-linq-провайдер) (8)
22. [Потокобезопасность и иммутабельность](#группа-22-потокобезопасность-и-иммутабельность) (6)
23. [Чек-лист лучших практик](#группа-23-чек-лист-лучших-практик) (5)

**Итого: 200 вопросов.**

### Общие источники (используются многократно ниже)

- MS Docs, серия «Expression Trees»: [Overview](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/) · [Explained](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-explained) · [Building](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-building) · [Execution](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-execution) · [Translating](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-translating) · [Interpreting](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-interpreting) · [Runtime support / API map](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-classes) · [Limitations](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/#limitations)
- API: [`Expression<TDelegate>`](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expression-1) · [`ExpressionVisitor`](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expressionvisitor) · [`ExpressionType`](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expressiontype) · [`IQueryable<T>`](https://learn.microsoft.com/dotnet/api/system.linq.iqueryable-1) · [`IQueryProvider`](https://learn.microsoft.com/dotnet/api/system.linq.iqueryprovider)
- Спецификация языка C#: [§8.6 Expression tree types](https://learn.microsoft.com/dotnet/csharp/language-reference/language-specification/types#86-expression-tree-types)
- Matt Warren, «LINQ: Building an IQueryable Provider» (классическая серия о том, как устроены LINQ-провайдеры): [Часть I](https://learn.microsoft.com/en-us/archive/blogs/mattwar/linq-building-an-iqueryable-provider-part-i), [список всех частей](https://learn.microsoft.com/en-us/archive/blogs/mattwar/linq-building-an-iqueryable-provider-series), продолжение проекта — [IQToolkit на GitHub](https://github.com/mattwar/iqtoolkit)
- EF Core: [How query processing works](https://learn.microsoft.com/ef/core/querying/how-query-works), [Dynamic queries via expression trees](https://learn.microsoft.com/dotnet/csharp/linq/how-to-build-dynamic-queries)
- Библиотеки: [System.Linq.Dynamic.Core](https://github.com/zzzprojects/System.Linq.Dynamic.Core) · [LINQKit / PredicateBuilder](https://github.com/scottksmith95/LINQKit) · [Moq (NuGet)](https://www.nuget.org/packages/Moq) · [AutoMapper (NuGet)](https://www.nuget.org/packages/AutoMapper) · [FluentValidation (NuGet)](https://www.nuget.org/packages/FluentValidation)

---
## Группа 1: Основы и концепция Expression Trees

### 1. [I] Что такое Expression Tree и зачем оно нужно?

**Ответ.** Expression Tree («дерево выражений») — это структура данных, представляющая код (обычно — лямбда-выражение) не как исполняемые инструкции, а как граф объектов-узлов, каждый из которых описывает одну синтаксическую конструкцию: константу, бинарную операцию, вызов метода, обращение к свойству и т.д. В отличие от делегата (`Func<T>`), который компилятор превращает сразу в IL, expression tree можно **исследовать во время выполнения**: обойти узлы, прочитать их семантику, изменить и только потом (по желанию) скомпилировать в исполняемый делегат.

Практическая ценность: expression trees позволяют библиотеке получить не «чёрный ящик»-функцию, а её описание. Именно так Entity Framework превращает `.Where(x => x.Age > 18)` в SQL `WHERE Age > 18`, а не выполняет C#-код построчно — потому что видит *структуру* выражения, а не скомпилированный IL.

**Диаграмма.**
```
Код:      x => x.Age > 18
Дерево:
        LambdaExpression
             |
        BinaryExpression (GreaterThan)
          /              \
  MemberExpression      ConstantExpression
   (x.Age)                  (18)
      |
 ParameterExpression (x)
```

**Ресурсы:** [MS Docs: Expression Trees — Overview](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/) · [Expression trees — data that defines code](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-explained)

---

### 2. [I] Чем Expression Tree принципиально отличается от делегата (`Func`/`Action`)?

**Ответ.** Делегат — это готовый, непрозрачный указатель на исполняемый код (метод или скомпилированную лямбду): вы можете его вызвать, но не можете «заглянуть внутрь» и узнать, что именно он делает. Expression tree — это **данные о коде**: дерево объектов `Expression`, `MemberExpression`, `MethodCallExpression` и т.п., которые можно программно анализировать, изменять и транслировать в другое представление (например, в SQL или в другое дерево). Delegate = «выполни», Expression Tree = «пойми и, при желании, выполни». Именно эта прозрачность лежит в основе `IQueryable<T>`: `IEnumerable<T>` работает с делегатами (LINQ to Objects — исполняется в памяти), а `IQueryable<T>` работает с expression trees (LINQ to Entities/SQL — транслируется провайдером).

**Ресурсы:** [MS Docs: Expression Trees](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/) · [C# Language Spec §8.6](https://learn.microsoft.com/dotnet/csharp/language-reference/language-specification/types#86-expression-tree-types)

---

### 3. [I] Кто создаёт expression tree — компилятор или CLR?

**Ответ.** Expression tree целиком создаётся **компилятором C#** во время компиляции, а не CLR во время выполнения. Когда лямбда-выражение присваивается переменной типа `Expression<TDelegate>`, компилятор вместо генерации IL-кода тела лямбды генерирует последовательность вызовов фабричных методов класса `Expression` (`Expression.Parameter`, `Expression.Constant`, `Expression.GreaterThan`, `Expression.Lambda` и т.д.), которые во время выполнения программы построят объектный граф дерева. Это чисто библиотечный механизм (`System.Linq.Expressions`) — никакой специальной поддержки CLR/JIT не требуется вплоть до момента вызова `Compile()`, который уже использует `System.Reflection.Emit`/`DynamicMethod` под капотом.

**Ресурсы:** [MS Docs: Build expression trees](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-building)

---

### 4. [I] Приведите три реальных примера использования Expression Trees в популярных .NET-библиотеках.

**Ответ.**
1. **Entity Framework Core / EF6 / LINQ to SQL** — переводят `IQueryable<T>`-запросы в SQL, анализируя дерево выражений.
2. **Moq** — `mock.Setup(x => x.DoWork(It.IsAny<int>()))`: аргумент `Setup` — expression tree, из которой Moq извлекает, какой метод и с какими параметрами настраивается (в отличие от делегата, который бы просто выполнился).
3. **AutoMapper** — при создании карт (`CreateMap`) и особенно при `ProjectTo<TDto>()` строит expression tree, транслируемую в SQL-проекцию, чтобы не тянуть из БД лишние поля.

Другие примеры: FluentValidation (`RuleFor(x => x.Email)`), MediatR/AutoQuery, ASP.NET Core `[Route]`-подобные конструкторы, ORM-мапперы (Dapper.Contrib), сериализаторы конфигурации, спецификации (Specification pattern) в DDD.

**Ресурсы:** [MS Docs Overview — примеры EF/Moq](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/) · [Moq (NuGet)](https://www.nuget.org/packages/Moq)

---

### 5. [I] Что означает утверждение «Expression Trees неизменяемы (immutable)»?

**Ответ.** Каждый узел `Expression` — это неизменяемый (readonly) объект: после создания его нельзя «отредактировать на месте» (нет сеттеров для `Left`/`Right`/`Body` и т.д.). Чтобы «изменить» дерево, нужно построить **новое** дерево, переиспользуя часть старых узлов и подставляя новые там, где требуется изменение — это и делает `ExpressionVisitor`. Такой дизайн даёт безопасное разделение узлов между разными деревьями (один и тот же `ConstantExpression` можно переиспользовать в нескольких деревьях без риска побочных эффектов) и делает деревья потокобезопасными для чтения из нескольких потоков одновременно.

**Ресурсы:** [MS Docs: Immutability](https://learn.microsoft.com/dotnet/visual-basic/programming-guide/concepts/expression-trees/#immutability-of-expression-trees) · [Translate expression trees](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-translating)

---

### 6. [I] Какой namespace и сборка отвечают за Expression Trees?

**Ответ.** Основные типы находятся в пространстве имён `System.Linq.Expressions` (сборка `System.Linq.Expressions.dll`, начиная с .NET Core это отдельная сборка, входящая в netstandard-набор). Ключевые типы: абстрактный базовый класс `Expression`, обобщённый `Expression<TDelegate>` (наследник `LambdaExpression`), перечисление `ExpressionType`, класс-посетитель `ExpressionVisitor`, и десятки конкретных узлов (`BinaryExpression`, `UnaryExpression`, `MethodCallExpression`, `MemberExpression`, `ConditionalExpression`, `NewExpression`, `ParameterExpression`, `ConstantExpression`, `LambdaExpression`, `InvocationExpression`, `IndexExpression`, `ListInitExpression`, `MemberInitExpression`, `BlockExpression`, `LoopExpression`, `SwitchExpression`, `TryExpression`, `GotoExpression`, `LabelExpression`, `DynamicExpression`, `TypeBinaryExpression`, `DefaultExpression`).

**Ресурсы:** [API: System.Linq.Expressions](https://learn.microsoft.com/dotnet/api/system.linq.expressions)

---

### 7. [A] Как связаны Expression Trees и Dynamic Language Runtime (DLR)?

**Ответ.** DLR использует expression trees как **промежуточное представление (IR)** между динамическими языками (IronPython, IronRuby) и .NET. Вместо того чтобы компиляторы этих языков генерировали IL напрямую, они строят expression trees (в том числе с узлами `DynamicExpression`, представляющими динамическую операцию, разрешаемую через `CallSiteBinder` во время выполнения), которые затем компилируются в делегаты рантаймом DLR. Это позволило переиспользовать один и тот же механизм генерации кода для множества языков и дало C#/VB возможность использовать `dynamic` через тот же инфраструктурный слой (`Microsoft.CSharp.RuntimeBinder`).

**Ресурсы:** [MS Docs: Dynamic Language Runtime Overview](https://learn.microsoft.com/dotnet/framework/reflection-and-codedom/dynamic-language-runtime-overview) · [MS Docs Overview, раздел про DLR](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/)

---

### 8. [I] Может ли компилятор построить expression tree из произвольного блока кода (`{ }`)?

**Ответ.** Нет — компилятор C# строит expression tree **только из expression-лямбды** (однострочного выражения без тела в фигурных скобках, без `return`, без операторов). Из **statement-лямбды** (`x => { ... }`, с блоком, циклами, `if`, множественными операторами) компилятор дерево построить не может — это ограничение именно синтаксического преобразования компилятором, а не самого API `System.Linq.Expressions` (которое как раз поддерживает `BlockExpression`, `LoopExpression`, `ConditionalExpression` и т.д. — но собрать их вручную можно только через прямые вызовы `Expression.*`).

```csharp
Expression<Func<int,int>> ok = x => x + 1;          // компилируется
Expression<Func<int,int>> bad = x => { return x+1; }; // CS0834: statement lambda не может быть деревом
```

**Ресурсы:** [MS Docs: Limitations](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/#limitations)

---

### 9. [I] Что произойдёт, если попытаться скомпилировать statement-лямбду в `Expression<TDelegate>`?

**Ответ.** Это ошибка компиляции **CS0834**: *«A lambda expression with a statement body cannot be converted to an expression tree»*. Ошибка возникает на этапе компиляции C#, а не во время выполнения — компилятор просто не умеет транслировать операторы (`if`, `for`, присваивания, блоки) в узлы Expression API автоматически. Обойти это можно только вручную построив дерево через `Expression.Block`, `Expression.IfThenElse`, `Expression.Loop` и т.д.

**Ресурсы:** [MS Docs: Limitations](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/#limitations) · [MS Docs: Build expression trees — управляющие конструкции](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-building)

---

### 10. [I] Из чего состоит корневой класс `Expression` и что такое `ExpressionType`?

**Ответ.** `Expression` — абстрактный базовый класс всех узлов дерева. У него есть свойства `NodeType` (значение перечисления `ExpressionType`, например `Add`, `Call`, `MemberAccess`, `Lambda`) и `Type` (`System.Type` — тип значения, которое вернёт вычисление этого узла, например `typeof(bool)` для `x.Age > 18`). `ExpressionType` — это «тег», по которому удобно делать `switch`/`if` при обходе дерева без приведения типов, хотя в современном коде чаще применяют паттерн-матчинг по конкретному подтипу (`is BinaryExpression be`) или переопределение методов `ExpressionVisitor`.

**Ресурсы:** [API: Expression Class](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expression) · [API: ExpressionType Enum](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expressiontype)

---
## Группа 2: `Expression<TDelegate>` vs `Func`/`Action`, лямбды

### 11. [I] Что произойдёт при присваивании `x => x + 1` переменной `Func<int,int>` и переменной `Expression<Func<int,int>>`?

**Ответ.** Синтаксически лямбда одинакова, но компилятор генерирует для неё **два совершенно разных результата** в зависимости от целевого типа (target typing):

```csharp
Func<int,int> del = x => x + 1;               // компилятор эмитит IL-метод + делегат на него
Expression<Func<int,int>> exp = x => x + 1;    // компилятор эмитит код, СТРОЯЩИЙ дерево через Expression.*
```

Для `del` компилятор создаёт (обычно статический, если нет захвата состояния) метод с телом `return x + 1;` и делегат, ссылающийся на него — вызов `del(5)` напрямую исполняет IL. Для `exp` компилятор генерирует вызовы `Expression.Parameter`, `Expression.Add`, `Expression.Constant`, `Expression.Lambda<Func<int,int>>` — результат представляет код `x => x + 1` как данные; чтобы выполнить его, нужно сначала `exp.Compile()`.

**Ресурсы:** [C# Language Spec §8.6](https://learn.microsoft.com/dotnet/csharp/language-reference/language-specification/types#86-expression-tree-types) · [MS Docs Overview](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/)

---

### 12. [I] Почему такая перегрузка допустима: `Where(Func<T,bool>)` (Enumerable) и `Where(Expression<Func<T,bool>>)` (Queryable) — как компилятор выбирает нужную?

**Ответ.** Разрешение перегрузки идёт по типу *приёмника*: если объект реализует `IEnumerable<T>` (но не `IQueryable<T>`), выбирается `Enumerable.Where`, принимающий делегат — компилятор транслирует лямбду в делегат. Если объект — `IQueryable<T>`, выбирается `Queryable.Where`, принимающий `Expression<Func<T,bool>>` — та же лямбда транслируется в дерево. Так один и тот же LINQ-синтаксис (`from x in source where ... select ...`) работает и «в памяти» (LINQ to Objects), и «на удалённом источнике» (LINQ to Entities), в зависимости исключительно от статического типа `source`.

**Ресурсы:** [MS Docs: IQueryable LINQ providers](https://learn.microsoft.com/dotnet/csharp/linq/#iqueryable-linq-providers) · [API: Queryable Class](https://learn.microsoft.com/dotnet/api/system.linq.queryable)

---

### 13. [I] Можно ли вызвать `Expression<Func<T,bool>>` напрямую, как метод?

**Ответ.** Нет. `Expression<TDelegate>` не реализует `Invoke` — это не делегат, а описание кода. Чтобы получить исполняемую версию, нужно явно вызвать `.Compile()`, который вернёт `TDelegate`, и уже его вызывать: `expr.Compile()(value)`. Частая ошибка начинающих — пытаться писать `predicate(item)`, когда `predicate` имеет тип `Expression<Func<T,bool>>`; компилятор не даст это сделать (`predicate` не является delegate-совместимым типом для оператора вызова).

**Ресурсы:** [MS Docs: Execute expression trees](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-execution)

---

### 14. [A] Что произойдёт при попытке передать метод-группу (method group) как `Expression<TDelegate>`?

**Ответ.** Метод-группа (`SomeStaticMethod` без вызова) **не может** быть неявно преобразована в `Expression<TDelegate>` — компилятор допускает такое преобразование только для лямбда-выражений (anonymous function expressions). Так же не подходят анонимные методы (`delegate(int x) { return x+1; }`), созданные ключевым словом `delegate`. Обойти это можно, обернув вызов метода в лямбду: `Expression<Func<int,int>> e = x => SomeStaticMethod(x);` — тогда дерево будет содержать `MethodCallExpression`, ссылающийся на этот метод через `MethodInfo`.

**Ресурсы:** [MS Docs: Limitations — method group expressions](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/#limitations)

---

### 15. [I] Чем отличается `LambdaExpression` от `Expression<TDelegate>`?

**Ответ.** `Expression<TDelegate>` — это обобщённый (generic) класс-наследник нетипизированного `LambdaExpression`. `LambdaExpression` используется, когда конкретный тип делегата неизвестен на этапе компиляции (например, при построении дерева динамически через `Expression.Lambda(body, parameters)` без указания типа делегата) — тогда компилятор сам подбирает подходящий тип делегата (`Func<...>`/`Action<...>` либо специальный сгенерированный тип). Для выполнения нетипизированного `LambdaExpression` нельзя напрямую вызвать делегат через `()` — приходится вызывать `Compile().DynamicInvoke(...)`, что медленнее прямого вызова из-за упаковки аргументов в `object[]` и рефлексии.

**Ресурсы:** [API: LambdaExpression](https://learn.microsoft.com/dotnet/api/system.linq.expressions.lambdaexpression) · [API: LambdaExpression.Compile](https://learn.microsoft.com/dotnet/api/system.linq.expressions.lambdaexpression.compile) · [API: Delegate.DynamicInvoke](https://learn.microsoft.com/dotnet/api/system.delegate.dynamicinvoke)

---

### 16. [A] Как получить `Expression<Action<T>>` (без возвращаемого значения) и чем он отличается от `Expression<Func<T,object>>`?

**Ответ.** `Expression<Action<T>>` строится из лямбды, тело которой — вызов метода/оператор без возвращаемого значения: `Expression<Action<Logger>> e = l => l.Log("hi");`. Внутри это тот же `LambdaExpression` с `Body`, представляющим `MethodCallExpression`, но `ReturnType` равен `void`. В отличие от `Expression<Func<T,object>>`, которое требует, чтобы тело возвращало значение, приводимое к `object` (и для `void`-методов это невозможно без искусственной обёртки). Именно `Expression<Action<T>>` используется в Moq для настройки вызовов методов без возвращаемого значения: `mock.Setup(x => x.DoWork())`.

**Ресурсы:** [API: Expression<TDelegate>](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expression-1) · [Moq (NuGet)](https://www.nuget.org/packages/Moq)

---

### 17. [I] Что содержит свойство `Parameters` у `LambdaExpression` и зачем оно нужно отдельно от `Body`?

**Ответ.** `LambdaExpression.Parameters` — это `ReadOnlyCollection<ParameterExpression>`, список формальных параметров лямбды (например, для `(x, y) => x + y` — два `ParameterExpression`: `x` и `y`). `Body` — выражение, представляющее тело. Они хранятся раздельно, потому что при компиляции/трансляции дерева нужно точно знать, какие узлы `ParameterExpression` внутри `Body` являются «входными данными» лямбды, а не свободными переменными (замыканиями) — ссылки на параметры и ссылки на захваченные переменные визуально неотличимы без этого списка.

**Ресурсы:** [API: LambdaExpression.Parameters](https://learn.microsoft.com/dotnet/api/system.linq.expressions.lambdaexpression.parameters)

---

### 18. [A] Как выразить лямбду с несколькими параметрами через API вручную и как позже добавить типизацию через `Expression.Lambda<TDelegate>`?

**Ответ.** Общий метод `Expression.Lambda(body, parameters)` возвращает нетипизированный `LambdaExpression`, автоматически подбирая тип делегата по числу и типам параметров и типу `body`. Если нужен конкретный, заранее известный тип делегата — используют обобщённую перегрузку `Expression.Lambda<TDelegate>(body, parameters)`, которая сразу возвращает `Expression<TDelegate>` и на этапе построения проверяет совместимость сигнатур (иначе бросит `ArgumentException`, если, например, число параметров не совпадает с `TDelegate`).

```csharp
var x = Expression.Parameter(typeof(int), "x");
var y = Expression.Parameter(typeof(int), "y");
var body = Expression.Add(x, y);
Expression<Func<int,int,int>> add =
    Expression.Lambda<Func<int,int,int>>(body, x, y);
```

**Ресурсы:** [API: Expression.Lambda Method](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expression.lambda) · [MS Docs: Build expression trees](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-building)

---
## Группа 3: Анатомия узлов дерева (типы Expression)

### 19. [I] Что такое `ParameterExpression` и почему важна ссылочная идентичность (identity), а не просто имя?

**Ответ.** `ParameterExpression` представляет параметр лямбды или локальную переменную (в `BlockExpression`). Критически важно: два `ParameterExpression` с одинаковым именем и типом — это **разные объекты**, если не переиспользован один и тот же экземпляр. Дерево связывает использование параметра с его объявлением по **ссылке на объект**, а не по имени (имя вообще опционально и используется только для отладочного вывода). Поэтому при построении дерева вручную нужно один раз создать `var x = Expression.Parameter(typeof(int), "x")` и переиспользовать переменную `x` везде, где параметр упоминается — иначе `Expression.Lambda` не свяжет использования с объявлением и бросит `InvalidOperationException` («variable 'x' of type 'System.Int32' referenced from scope '', but it is not defined»).

**Ресурсы:** [API: ParameterExpression](https://learn.microsoft.com/dotnet/api/system.linq.expressions.parameterexpression) · [MS Docs: Build expression trees](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-building)

---

### 20. [I] Чем отличаются `ConstantExpression` и `MemberExpression` при захвате внешней переменной в лямбде?

**Ответ.** Это частый источник путаницы. `ConstantExpression` — это буквальный литерал константы, известный **на этапе построения дерева** (например, `5`, `"abc"`, либо значение, «замороженное» компилятором). `MemberExpression` — обращение к полю/свойству через выражение (`x.Age`, `obj.Field`). Когда лямбда захватывает локальную переменную из внешнего метода, компилятор C# **не** вставляет `ConstantExpression` со значением переменной напрямую (кроме простых случаев вроде static readonly полей в некоторых версиях) — вместо этого он создаёт скрытый класс-замыкание (closure class) с полем для этой переменной, а в дереве появляется `MemberExpression`, где `Expression` — это `ConstantExpression`, содержащий *экземпляр объекта-замыкания*, а `Member` — `FieldInfo` захваченного поля. Это объясняет, почему при `.ToString()` замкнутой лямбды видно что-то вроде `value(Namespace.Class+<>c__DisplayClass0_0).capturedVar`.

**Ресурсы:** [API: MemberExpression](https://learn.microsoft.com/dotnet/api/system.linq.expressions.memberexpression) · [API: ConstantExpression](https://learn.microsoft.com/dotnet/api/system.linq.expressions.constantexpression)

---

### 21. [I] Что представляет `BinaryExpression` и какие операции он покрывает?

**Ответ.** `BinaryExpression` — узел с двумя операндами (`Left`, `Right`) и оператором, закодированным в `NodeType` (`Add`, `Subtract`, `Multiply`, `Divide`, `Modulo`, `And`/`AndAlso`, `Or`/`OrElse`, `Equal`, `NotEqual`, `GreaterThan`, `LessThanOrEqual`, `Coalesce`, `ArrayIndex`, присваивание `Assign` и составные `AddAssign` и т.п.). Также содержит `Method` (перегруженный оператор, если применим, например пользовательский `operator +`) и `Conversion` (для `Coalesce` с явным преобразованием типов). Важно различать `And`/`Or` (небитовые/битовые логические без короткого замыкания) и `AndAlso`/`OrElse` (аналог `&&`/`||` с коротким замыканием) — они генерируют разные узлы `NodeType`.

**Ресурсы:** [API: BinaryExpression](https://learn.microsoft.com/dotnet/api/system.linq.expressions.binaryexpression)

---

### 22. [I] Что представляет `UnaryExpression` и какие операции в него входят помимо `-x`?

**Ответ.** `UnaryExpression` — узел с одним операндом (`Operand`). Помимо очевидного унарного минуса (`Negate`), сюда относятся: `Convert`/`ConvertChecked` (приведение типов, в т.ч. упаковка/распаковка), `Not` (логическое/битовое отрицание), `TypeAs` (оператор `as`), `Quote` (специальный узел, оборачивающий вложенное лямбда-выражение внутри другого дерева — важно для nested LINQ-запросов), `ArrayLength`, `Increment`/`Decrement`, `Throw`, `UnaryPlus`. Наличие отдельного `Convert`-узла — то, почему `x => (double)x.IntValue` в дереве выглядит как `UnaryExpression(Convert, MemberExpression(x.IntValue))`.

**Ресурсы:** [API: UnaryExpression](https://learn.microsoft.com/dotnet/api/system.linq.expressions.unaryexpression)

---

### 23. [A] Что такое узел `Quote` (`ExpressionType.Quote`) и когда он появляется?

**Ответ.** `Quote` — специальный `UnaryExpression`, который появляется, когда внутри одного expression tree встречается **другое** лямбда-выражение как аргумент (типичный случай — вложенные LINQ-запросы через `IQueryable`, например `orders.Select(o => o.Items.Where(i => i.Qty > 1))`). Внутренняя лямбда `i => i.Qty > 1` сама по себе не «выполняется» и не «встраивается» напрямую — она оборачивается в `Quote`, чтобы дерево осталось деревом (данными), а не превращалось в скомпилированный делегат раньше времени. Провайдер (например EF) разворачивает `Quote`, извлекая `Operand` — исходную `LambdaExpression`, — и рекурсивно транслирует её.

**Ресурсы:** [API: ExpressionType.Quote](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expressiontype) · [MS Docs: Expression classes](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-classes)

---

### 24. [I] Чем отличаются `MethodCallExpression.Object` и `MethodCallExpression.Arguments`, и как представлены статические методы?

**Ответ.** `MethodCallExpression` описывает вызов метода: `Object` — выражение-приёмник (например, `x` в `x.ToString()`), `Method` — `MethodInfo` вызываемого метода, `Arguments` — список выражений-аргументов. Для **статических** методов (в т.ч. методов расширения, включая сами `Where`/`Select`/`OrderBy` из `Queryable`) `Object` равен `null`, а сам объект-приёмник (если это метод расширения) передаётся как первый элемент `Arguments`. Именно поэтому цепочка `.Where(...).Select(...)` в LINQ to Queryable — это вложенные `MethodCallExpression`, где каждый следующий вызов оборачивает предыдущий как первый аргумент.

**Ресурсы:** [API: MethodCallExpression](https://learn.microsoft.com/dotnet/api/system.linq.expressions.methodcallexpression)

---

### 25. [I] Что такое `NewExpression` и `MemberInitExpression`, чем они различаются?

**Ответ.** `NewExpression` представляет вызов конструктора (`new Person(name, age)`) — содержит `Constructor` (`ConstructorInfo`) и `Arguments`. `MemberInitExpression` представляет создание объекта **с инициализатором членов** (`new Person { Name = name, Age = age }`) — содержит `NewExpression` (вызов конструктора, часто беспараметрического) плюс коллекцию `Bindings` (`MemberBinding`), описывающую какие свойства/поля чем инициализируются. Это различие критично для ORM-проекций: EF Core транслирует `Select(x => new Dto { Id = x.Id, Name = x.Name })` именно через анализ `MemberInitExpression.Bindings`, формируя список выбираемых колонок.

**Ресурсы:** [API: NewExpression](https://learn.microsoft.com/dotnet/api/system.linq.expressions.newexpression) · [API: MemberInitExpression](https://learn.microsoft.com/dotnet/api/system.linq.expressions.memberinitexpression)

---

### 26. [A] Что такое `ListInitExpression` и когда компилятор его генерирует?

**Ответ.** `ListInitExpression` представляет коллекционный инициализатор — `new List<int> { 1, 2, 3 }` или `new Dictionary<string,int> { ["a"] = 1 }`. Он содержит `NewExpression` (вызов конструктора коллекции) и коллекцию `ElementInit` — каждый элемент описывает вызов метода добавления (`Add`, `AddRange` и т.д., через `MethodInfo`) с соответствующими аргументами. Отличие от `NewArrayInit` (создание массива `new[] {1,2,3}`, отдельный тип узла): `ListInitExpression` подразумевает вызов метода `Add` для каждого элемента, тогда как `NewArrayInit` — это прямая инициализация элементов массива без вызовов методов.

**Ресурсы:** [API: ListInitExpression](https://learn.microsoft.com/dotnet/api/system.linq.expressions.listinitexpression) · [API: NewArrayExpression](https://learn.microsoft.com/dotnet/api/system.linq.expressions.newarrayexpression)

---

### 27. [A] Что представляет `TypeBinaryExpression` и в каком контексте он используется?

**Ответ.** `TypeBinaryExpression` представляет проверку типа во время выполнения — оператор `is` (`ExpressionType.TypeIs`) или (с C# 7+ и `Expression.TypeEqual`) `ExpressionType.TypeEqual`. Содержит `Expression` (проверяемое выражение) и `TypeOperand` (`System.Type`, с которым сравнивается). Пример: `x => x is Manager` строится как `TypeBinaryExpression { Expression = x, TypeOperand = typeof(Manager) }`. EF Core активно использует этот узел при трансляции иерархий наследования (TPH/TPT) в SQL с проверкой дискриминатора.

**Ресурсы:** [API: TypeBinaryExpression](https://learn.microsoft.com/dotnet/api/system.linq.expressions.typebinaryexpression)

---

### 28. [A] Чем `InvocationExpression` отличается от `MethodCallExpression`?

**Ответ.** `InvocationExpression` представляет **вызов делегата/лямбда-выражения** как значения (`Expression`), а не вызов именованного метода через `MethodInfo`. Он создаётся вызовом `Expression.Invoke(lambdaOrDelegateExpr, arguments)` и означает «выполнить то выражение, которое даёт делегат, с этими аргументами» — например, для инлайнинга одного дерева в другое: `Expression.Invoke(innerLambda, x)` вставляет вызов `innerLambda(x)` внутрь внешнего дерева. Это именно тот механизм, на котором строится `LINQKit.Invoke()` и `PredicateBuilder`, позволяющие комбинировать независимо построенные `Expression<Func<T,bool>>` — но многие LINQ-провайдеры (включая ранние версии EF) **не умеют** транслировать `InvocationExpression` напрямую, отсюда необходимость «разворачивать» (expand) его через специальный `ExpressionVisitor` перед выполнением запроса.

**Ресурсы:** [API: InvocationExpression](https://learn.microsoft.com/dotnet/api/system.linq.expressions.invocationexpression) · [LINQKit](https://github.com/scottksmith95/LINQKit)

---

### 29. [A] Что такое `IndexExpression` и чем он отличается от `Expression.ArrayIndex`?

**Ответ.** `IndexExpression` представляет обращение по индексатору — как к массиву (`arr[i]`), так и к индексатору свойства (`dict["key"]`, `list[0]` через `this[int]`). У него есть `Object` (приёмник), `Indexer` (`PropertyInfo` для пользовательских индексаторов, `null` для массивов) и `Arguments` (список индексов — поддерживает многомерные индексаторы `matrix[i,j]`). `Expression.ArrayIndex` — более старый, специализированный статический метод-фабрика, создающий `BinaryExpression` (для одномерного случая) или `MethodCallExpression` (для многомерных массивов) именно для доступа к элементам **массива**; `Expression.MakeIndex`/`Expression.Property(obj, indexer, args)` создают полноценный `IndexExpression`, который унифицированно работает и с массивами, и с индексаторами классов.

**Ресурсы:** [API: IndexExpression](https://learn.microsoft.com/dotnet/api/system.linq.expressions.indexexpression) · [API: Expression.ArrayIndex](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expression.arrayindex)

---

### 30. [G] Какие узлы отвечают за управляющие конструкции (циклы, блоки, switch, try/catch) и почему они «не как в C#»?

**Ответ.** API `System.Linq.Expressions` поддерживает управляющие конструкции на уровне более низком и универсальном, чем C#:
- `BlockExpression` (`Expression.Block`) — последовательность выражений с локальными переменными, аналог `{ ... }`.
- `LoopExpression` (`Expression.Loop`) — **единственный** вид цикла (нет отдельных `for`/`while`/`foreach`!), бесконечный цикл, из которого выходят через `GotoExpression` с `BreakLabel`; `ContinueLabel` реализует переход к следующей итерации.
- `ConditionalExpression` (`Expression.Condition`/`IfThen`/`IfThenElse`) — аналог `if`/тернарного оператора.
- `SwitchExpression` (`Expression.Switch`) — аналог `switch`.
- `TryExpression` (`Expression.TryCatch`/`TryFinally`) — аналог `try/catch/finally`.
- `GotoExpression` + `LabelExpression`/`LabelTarget` — низкоуровневые `goto`/`break`/`continue`/`return`.

Почему «один цикл»: разработчики API выбрали минимальный набор примитивов, из которых на уровне построителя дерева (не компилятора C#) можно собрать `for`/`while`/`foreach`, вместо того чтобы поддерживать все синтаксические варианты C# как отдельные типы узлов. Это удобно для генераторов кода (например, для ORM или интерпретаторов DSL), но недоступно из C#-лямбд напрямую — только через явное построение API.

**Диаграмма (эмуляция `while (cond) body`):**
```
LoopExpression
 └─ Body: BlockExpression
      ├─ IfThen(Not(cond), Goto(breakLabel))
      ├─ body
      └─ (неявный возврат к началу цикла)
 Label: breakLabel
```

**Ресурсы:** [MS Docs: Build expression trees — control flow](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-building) · [API: LoopExpression](https://learn.microsoft.com/dotnet/api/system.linq.expressions.loopexpression) · [API: TryExpression](https://learn.microsoft.com/dotnet/api/system.linq.expressions.tryexpression)

---

### 31. [A] Зачем нужен `DefaultExpression` и как он используется, например, для `default(T)`?

**Ответ.** `DefaultExpression` (создаётся через `Expression.Default(type)`) представляет значение по умолчанию для указанного типа — эквивалент `default(T)` в C#. Он также используется как «пустое» тело для узла `void`-типа (например, тело `Expression.Empty()` — частный случай `Default(typeof(void))`), когда нужно синтаксически указать «ничего не делать», сохраняя корректный тип узла в местах, где ожидается выражение (например, ветка `else` без действия в `IfThen`).

**Ресурсы:** [API: DefaultExpression](https://learn.microsoft.com/dotnet/api/system.linq.expressions.defaultexpression) · [API: Expression.Default](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expression.default)

---

### 32. [G] Что такое `DebugInfoExpression` и `DynamicExpression`?

**Ответ.** `DebugInfoExpression` (`Expression.DebugInfo`) привязывает диапазон исходного кода (файл, строки/столбцы) к узлу дерева — используется вместе с `SymbolDocumentInfo` и `DebugInfoGenerator`, чтобы при компиляции через `LambdaExpression.CompileToMethod` (только .NET Framework) сгенерированный метод можно было отлаживать построчно, как обычный C#-код. `DynamicExpression` (`Expression.Dynamic`) представляет операцию, разрешаемую динамически через `CallSiteBinder` во время выполнения — именно этот узел лежит в основе поддержки `dynamic` в C# и используется DLR-языками; он **не поддерживается** компилятором C# при построении `Expression<TDelegate>` из обычных лямбд (`dynamic`-операции — в списке ограничений).

**Ресурсы:** [API: DebugInfoExpression](https://learn.microsoft.com/dotnet/api/system.linq.expressions.debuginfoexpression) · [API: DynamicExpression](https://learn.microsoft.com/dotnet/api/system.linq.expressions.dynamicexpression) · [MS Docs: Limitations — dynamic operations](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/#limitations)

---

### 33. [I] Как безопасно (без ручного `switch` по всем 50+ типам) определить конкретный вид узла при обходе дерева?

**Ответ.** Три подхода:
1. **Паттерн-матчинг по типу C#**: `if (node is BinaryExpression be) { ... }` — самый читаемый способ для «точечного» анализа отдельных узлов.
2. **Переопределение виртуальных методов `ExpressionVisitor`**: `VisitBinary`, `VisitMethodCall`, `VisitMember` и т.д. — правильный способ для полного, рекурсивного обхода дерева, так как базовая реализация сама заботится о рекурсии в дочерние узлы.
3. **`switch` по `NodeType`** (`ExpressionType`) — полезен, когда одному C#-типу узла соответствует несколько разных операций (например, `BinaryExpression` может быть `Add`, `Subtract`, `Equal` и т.д.), и нужно различать именно семантику операции, а не просто C#-тип.

На практике для промышленного кода почти всегда выбирают вариант 2 (наследование от `ExpressionVisitor`), потому что он избавляет от необходимости вручную писать рекурсивный обход всех дочерних узлов.

**Ресурсы:** [API: ExpressionVisitor](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expressionvisitor)

---
## Группа 4: Построение деревьев вручную (Expression API)

### 34. [I] Почему деревья строятся «снизу вверх» (от листьев к корню), а не «сверху вниз»?

**Ответ.** Из-за неизменяемости: каждый узел создаётся с уже готовыми дочерними узлами, переданными в конструктор/фабричный метод (`Expression.Add(left, right)` требует, чтобы `left` и `right` уже существовали). Невозможно сначала создать «пустой» родительский узел и потом «вписать» в него детей — сеттеров нет. Поэтому типичный порядок: сначала листья (`Expression.Constant`, `Expression.Parameter`), затем промежуточные узлы (`Expression.Add`, `Expression.Call`), и в конце — `Expression.Lambda`, оборачивающий всё дерево целиком.

```csharp
var one = Expression.Constant(1);
var two = Expression.Constant(2);
var sum = Expression.Add(one, two);          // после one, two
var lambda = Expression.Lambda<Func<int>>(sum); // после sum
```

**Ресурсы:** [MS Docs: Build expression trees](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-building)

---

### 35. [I] Как построить дерево, эквивалентное `num => num < 5`, используя только API (без лямбда-синтаксиса)?

**Ответ.**
```csharp
ParameterExpression numParam = Expression.Parameter(typeof(int), "num");
ConstantExpression five = Expression.Constant(5, typeof(int));
BinaryExpression numLessThanFive = Expression.LessThan(numParam, five);
Expression<Func<int, bool>> lambda =
    Expression.Lambda<Func<int, bool>>(numLessThanFive, new[] { numParam });
```
Именно так этот пример дан в официальной документации Microsoft — он демонстрирует базовый цикл: параметр → константа → бинарное сравнение → обёртка в lambda с явным указанием списка параметров (список должен включать все `ParameterExpression`, использованные в теле, и именно те же объекты-ссылки).

**Ресурсы:** [MS Docs: Build expression trees — Map code constructs](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-building#map-code-constructs-to-expressions)

---

### 36. [I] Как построить вызов метода `Math.Sqrt` внутри дерева и почему нужен `typeof(Math).GetMethod(...)`?

**Ответ.** `Expression.Call` требует объект `MethodInfo`, описывающий целевой метод (для статических методов — без приёмника). Получить его можно через рефлексию:
```csharp
var xParam = Expression.Parameter(typeof(double), "x");
var yParam = Expression.Parameter(typeof(double), "y");
var sum = Expression.Add(Expression.Multiply(xParam, xParam), Expression.Multiply(yParam, yParam));
var sqrtMethod = typeof(Math).GetMethod(nameof(Math.Sqrt), new[] { typeof(double) })
    ?? throw new InvalidOperationException("Math.Sqrt not found");
var distance = Expression.Call(sqrtMethod, sum);
var lambda = Expression.Lambda<Func<double,double,double>>(distance, xParam, yParam);
```
`GetMethod` может вернуть `null`, если сигнатура указана неверно или сборка недоступна — поэтому всегда стоит проверять результат и явно бросать осмысленное исключение, а не получать `NullReferenceException` в `Expression.Call`.

**Ресурсы:** [MS Docs: Build expression trees — Build a tree](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-building#build-a-tree)

---

### 37. [I] Как построить обращение к свойству объекта (`x => x.Name`) вручную?

**Ответ.**
```csharp
var param = Expression.Parameter(typeof(Person), "x");
var property = Expression.Property(param, nameof(Person.Name)); // или через PropertyInfo
var lambda = Expression.Lambda<Func<Person,string>>(property, param);
```
`Expression.Property` возвращает `MemberExpression`. Использование `nameof(Person.Name)` вместо строкового литерала `"Name"` рекомендуется, поскольку это даёт проверку на этапе компиляции и безопасный рефакторинг (IDE переименует и строку).

**Ресурсы:** [API: Expression.Property Method](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expression.property)

---

### 38. [A] Как построить цепочку вызовов `Where().Select()` в виде дерева вручную (эмулируя `IQueryable`-запрос)?

**Ответ.** Нужно найти обобщённые методы `Queryable.Where`/`Queryable.Select`, сделать их конкретными через `MakeGenericMethod` и вызвать `Expression.Call` со статическим методом, где первый аргумент — исходный `Expression` (`ConstantExpression`, представляющий `IQueryable<T>`), второй — `Quote`-обёрнутая лямбда-предикат/селектор:
```csharp
var source = Expression.Constant(queryable);
var param = Expression.Parameter(typeof(Person), "p");
var predicate = Expression.Lambda<Func<Person,bool>>(
    Expression.GreaterThan(Expression.Property(param, "Age"), Expression.Constant(18)), param);

var whereCall = Expression.Call(
    typeof(Queryable), nameof(Queryable.Where), new[] { typeof(Person) },
    source, predicate);
```
Именно так работают внутри библиотеки типа AutoMapper `ProjectTo`, EF Core компоненты компиляции запросов и Dynamic LINQ.

**Ресурсы:** [API: Expression.Call Method](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expression.call) · [API: Queryable Class](https://learn.microsoft.com/dotnet/api/system.linq.queryable)

---

### 39. [A] Как построить дерево, вычисляющее факториал, учитывая, что нельзя использовать statement-лямбду и рекурсию по имени?

**Ответ.** Официальный MS-пример строит факториал через `LoopExpression` с `LabelTarget` для выхода и накопительной переменной, а не через рекурсию:
```csharp
var value = Expression.Parameter(typeof(int), "value");
var result = Expression.Variable(typeof(int), "result");
var breakLabel = Expression.Label(typeof(int), "break");

var loop = Expression.Block(
    new[] { result },
    Expression.Assign(result, Expression.Constant(1)),
    Expression.Loop(
        Expression.IfThenElse(
            Expression.GreaterThan(value, Expression.Constant(1)),
            Expression.Block(
                Expression.MultiplyAssign(result, value),
                Expression.PostDecrementAssign(value)),
            Expression.Break(breakLabel, result)),
        breakLabel));

var factorial = Expression.Lambda<Func<int,int>>(loop, value).Compile();
```
Рекурсивный вызов самого себя внутри expression tree в принципе невозможен для `Expression<TDelegate>`, построенного из C#-лямбды (нет имени, на которое можно сослаться до завершения построения) — для рекурсии внутри API нужен трюк с `Y`-комбинатором или предварительно объявленной переменной-делегатом, которой присваивают лямбду уже после её компиляции.

**Ресурсы:** [MS Docs: Interpret expressions — factorial example](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-interpreting#extending-this-sample) · [API: LoopExpression](https://learn.microsoft.com/dotnet/api/system.linq.expressions.loopexpression)

---

### 40. [I] Чем отличаются `Expression.Equal` и метод `.Equals()`, и почему для сравнения строк нужно быть внимательным?

**Ответ.** `Expression.Equal(left, right)` создаёт `BinaryExpression` с `NodeType.Equal`, что соответствует оператору `==` (и для строк это вызов `string.Equals` или сравнение ссылок в зависимости от типов операндов и наличия перегруженного `==`). Если нужно явно смоделировать вызов метода `object.Equals()`/`string.Equals(string, StringComparison)`, следует использовать `Expression.Call` с соответствующим `MethodInfo`, а не `Expression.Equal` — они порождают разные узлы (`BinaryExpression` vs `MethodCallExpression`), и LINQ-провайдеры могут транслировать их в разный SQL (например, `col = @p` против `col LIKE ...`/функции сравнения с учётом регистра).

**Ресурсы:** [API: Expression.Equal Method](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expression.equal)

---

### 41. [A] Как обработать ситуацию, когда типы операндов в `Expression.Add`/`Expression.Equal` не совпадают (например, `int` и `double`)?

**Ответ.** Большинство бинарных фабричных методов (`Add`, `Equal`, `GreaterThan`...) требуют, чтобы типы `Left` и `Right` были **совместимы** (совпадали или существовала предопределённая/пользовательская операция для такой пары); иначе выбрасывается `InvalidOperationException` («The binary operator Equal is not defined for the types ...»). Решение — явно вставить `Expression.Convert(node, targetType)` для приведения одного из операндов к типу другого перед созданием бинарного узла:
```csharp
var left = Expression.Property(param, "IntValue");   // int
var right = Expression.Constant(3.14);                // double
var leftConverted = Expression.Convert(left, typeof(double));
var comparison = Expression.Equal(leftConverted, right);
```

**Ресурсы:** [API: Expression.Convert Method](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expression.convert) · [API: BinaryExpression](https://learn.microsoft.com/dotnet/api/system.linq.expressions.binaryexpression)

---

### 42. [I] Как построить условное выражение (`x > 0 ? "positive" : "non-positive"`) через API?

**Ответ.**
```csharp
var x = Expression.Parameter(typeof(int), "x");
var condition = Expression.GreaterThan(x, Expression.Constant(0));
var conditional = Expression.Condition(
    condition,
    Expression.Constant("positive"),
    Expression.Constant("non-positive"));
var lambda = Expression.Lambda<Func<int,string>>(conditional, x);
```
`Expression.Condition` требует, чтобы типы веток `IfTrue` и `IfFalse` совпадали (или один из них — `void`-выражение при использовании `IfThen`/`IfThenElse` без возвращаемого значения); если типы разные, но приводимые — нужно явно обернуть в `Expression.Convert`, автоматического приведения типов здесь нет (в отличие от тернарного оператора C#, где компилятор сам находит общий тип).

**Ресурсы:** [API: Expression.Condition Method](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expression.condition)

---

### 43. [A] Как построить создание объекта с инициализацией свойств (`new Person { Name = "A", Age = 20 }`) через API?

**Ответ.**
```csharp
var ctor = typeof(Person).GetConstructor(Type.EmptyTypes)!;
var nameProp = typeof(Person).GetProperty(nameof(Person.Name))!;
var ageProp = typeof(Person).GetProperty(nameof(Person.Age))!;

var memberInit = Expression.MemberInit(
    Expression.New(ctor),
    Expression.Bind(nameProp, Expression.Constant("A")),
    Expression.Bind(ageProp, Expression.Constant(20)));

var lambda = Expression.Lambda<Func<Person>>(memberInit);
```
`Expression.Bind` создаёт `MemberAssignment` — самый частый вид `MemberBinding`; для вложенной инициализации коллекций/объектов есть также `Expression.MemberBind` (вложенный `MemberInit`) и `Expression.ListBind` (вложенная инициализация коллекции).

**Ресурсы:** [API: Expression.MemberInit Method](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expression.memberinit) · [API: Expression.Bind Method](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expression.bind)

---

### 44. [G] Как избежать «магических строк» при рефлексии внутри построителей выражений (`GetMethod`, `GetProperty`)?

**Ответ.** Несколько практик:
1. Использовать `nameof(...)` вместо строковых литералов там, где это возможно (для свойств/полей — почти всегда).
2. Для методов с перегрузками — извлекать `MethodInfo` через типизированное `Expression<Action<T>>`/`Expression<Func<T,TResult>>` и `ExpressionVisitor`/`(body as MethodCallExpression)?.Method`, а не строить сигнатуру вручную:
```csharp
static MethodInfo GetMethodInfo<T>(Expression<Action<T>> expr) =>
    ((MethodCallExpression)expr.Body).Method;

var containsMethod = GetMethodInfo<string>(s => s.Contains(""));
```
Это гарантирует, что при рефакторинге сигнатуры метода код построения дерева не «молча» сломается на этапе выполнения (`GetMethod` вернёт `null`), а даст ошибку компиляции.
3. Кэшировать полученные `MethodInfo`/`PropertyInfo` в статических полях — рефлексионные запросы недёшевы при частом построении деревьев в горячем пути.

**Ресурсы:** [API: MethodInfo](https://learn.microsoft.com/dotnet/api/system.reflection.methodinfo) · [MS Docs: Build expression trees](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-building)

---

### 45. [A] Чем отличается `Expression.PropertyOrField` от явных `Expression.Property`/`Expression.Field`, и когда стоит их избегать?

**Ответ.** `Expression.PropertyOrField(instance, "Name")` ищет член по имени сначала среди свойств, потом среди полей, и создаёт соответствующий `MemberExpression` — удобно для универсальных «построителей по имени», получающих имя члена как строку во время выполнения (например, для динамических предикатов из UI-фильтра). Недостаток: ошибки опечаток в имени обнаруживаются только в рантайме (`ArgumentException`, «Instance property or field ... is not defined»), а не в компиляции — поэтому в статически известных сценариях предпочтительнее явные `Expression.Property`/`Expression.Field` с `PropertyInfo`/`FieldInfo`, полученными типобезопасно (см. вопрос 44).

**Ресурсы:** [API: Expression.PropertyOrField Method](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expression.propertyorfield)

---
## Группа 5: ExpressionVisitor: обход и трансформация

### 46. [I] Что такое `ExpressionVisitor` и зачем наследоваться от него вместо ручного обхода?

**Ответ.** `ExpressionVisitor` — абстрактный базовый класс из `System.Linq.Expressions`, реализующий паттерн Visitor для деревьев выражений: у него есть виртуальные методы `VisitBinary`, `VisitUnary`, `VisitMethodCall`, `VisitMember`, `VisitConstant`, `VisitParameter`, `VisitConditional`, `VisitLambda<T>` и т.д. — по одному на каждый тип узла, плюс общий диспетчер `Visit(Expression)`, который определяет реальный тип узла и вызывает соответствующий специализированный метод. Базовая реализация каждого `VisitXxx` уже умеет рекурсивно посещать все дочерние узлы и (если что-то изменилось) собирать **новый** узел того же типа — благодаря этому, переопределив только нужные методы, можно как читать дерево, так и трансформировать его, не заботясь вручную о полном обходе всех 50+ типов узлов.

**Ресурсы:** [API: ExpressionVisitor](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expressionvisitor) · [MS Docs: Translate expression trees](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-translating)

---

### 47. [I] Классический пример: как заменить `AndAlso` (`&&`) на `OrElse` (`||`) во всём дереве?

**Ответ.**
```csharp
public class AndAlsoModifier : ExpressionVisitor
{
    public Expression Modify(Expression expression) => Visit(expression);

    protected override Expression VisitBinary(BinaryExpression b)
    {
        if (b.NodeType == ExpressionType.AndAlso)
        {
            var left = Visit(b.Left);
            var right = Visit(b.Right);
            return Expression.OrElse(left, right);
        }
        return base.VisitBinary(b);
    }
}
```
Это ровно пример из официальной документации Microsoft. Обратите внимание: **важно** рекурсивно вызывать `Visit` для `Left`/`Right` перед сборкой нового узла — иначе вложенные `AndAlso` внутри поддеревьев останутся незамеченными.

**Ресурсы:** [MS Docs: How to modify expression trees](https://learn.microsoft.com/dotnet/visual-basic/programming-guide/concepts/expression-trees/how-to-modify-expression-trees) · [MS Docs: Translate expression trees](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-translating#traverse-and-execute-an-addition)

---

### 48. [I] Почему при переопределении `VisitXxx` **обязательно** нужно вызывать `base.VisitXxx(node)` для необрабатываемых случаев, а не просто `return node`?

**Ответ.** `return node` «на всякий случай» кажется безопасным, но ломает трансформацию, если изменяемый узел находится **глубже** внутри поддерева этого типа. Например, если `VisitBinary` для не-`AndAlso` случаев просто вернёт `node` без изменений, а внутри `node.Left` скрывается ещё один `BinaryExpression` с `AndAlso`, требующий замены — этот вложенный узел никогда не будет посещён, потому что рекурсия остановится. `base.VisitBinary(node)` вместо этого рекурсивно вызывает `Visit` для всех дочерних выражений и, если хоть один дочерний узел изменился, создаёт новый узел этого же типа с обновлёнными детьми (а если ничего не изменилось — возвращает тот же объект `node`, что экономит аллокации).

**Ресурсы:** [API: ExpressionVisitor.VisitBinary](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expressionvisitor.visitbinary) · [MS Docs: Translate expression trees](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-translating)

---

### 49. [I] Как написать визитор, который вычисляет (интерпретирует) простое арифметическое дерево без компиляции?

**Ответ.** Можно наследоваться от `ExpressionVisitor`, но так как базовый класс возвращает `Expression`, для чистого «интерпретатора со значением» удобнее написать собственный рекурсивный метод (не обязательно наследовать `ExpressionVisitor`, если задача — не трансформация, а именно вычисление):
```csharp
static int Evaluate(Expression expr) => expr switch
{
    ConstantExpression c => (int)c.Value!,
    BinaryExpression { NodeType: ExpressionType.Add } b => Evaluate(b.Left) + Evaluate(b.Right),
    BinaryExpression { NodeType: ExpressionType.Multiply } b => Evaluate(b.Left) * Evaluate(b.Right),
    _ => throw new NotSupportedException(expr.NodeType.ToString())
};
```
Это упрощённая версия того, что делает `Compile()` внутри, только без генерации IL — просто прямая рекурсивная интерпретация дерева. Полезно для DSL с очень ограниченным набором операций, где генерация IL избыточна.

**Ресурсы:** [MS Docs: Interpret expressions](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-interpreting)

---

### 50. [A] Как реализовать визитор, заменяющий один `ParameterExpression` на конкретное значение/другое выражение («параметрическая подстановка»)?

**Ответ.** Классический `ReplaceParameterVisitor` (концептуально похож на `ReplacingExpressionVisitor` из внутреннего кода EF Core):
```csharp
public class ReplaceParameterVisitor : ExpressionVisitor
{
    private readonly ParameterExpression _source;
    private readonly Expression _target;
    public ReplaceParameterVisitor(ParameterExpression source, Expression target)
        { _source = source; _target = target; }

    protected override Expression VisitParameter(ParameterExpression node) =>
        node == _source ? _target : base.VisitParameter(node);
}
```
Это базовый строительный блок для комбинирования двух независимо построенных `Expression<Func<T,bool>>` (например, в `PredicateBuilder`/LINQKit): нужно взять тело одного выражения и «подставить» в него параметр другого, чтобы оба выражения ссылались на один и тот же `ParameterExpression`.

**Ресурсы:** [API: ReplacingExpressionVisitor (EF Core, для справки о паттерне)](https://learn.microsoft.com/dotnet/api/microsoft.entityframeworkcore.query.replacingexpressionvisitor) · [LINQKit](https://github.com/scottksmith95/LINQKit)

---

### 51. [G] Что произойдёт, если в переопределённом `VisitMethodCall` вернуть узел другого типа (не `MethodCallExpression`), и допустимо ли это?

**Ответ.** Да, это допустимо и является распространённой техникой — сигнатура `protected override Expression VisitMethodCall(MethodCallExpression node)` возвращает базовый тип `Expression`, поэтому визитор может заменить вызов метода на выражение совершенно другого вида, например заменить `x.ToUpper().Contains("A")` целиком на `ConstantExpression(true)` при определённых условиях (частичное вычисление/оптимизация запроса), либо развернуть `InvocationExpression`, встретив вызов «квотированной» лямбды. Единственное строгое требование — `Type` возвращаемого выражения должен быть совместим с контекстом, в котором стоял заменяемый узел (иначе выше по дереву при сборке родительского узла или на этапе `Compile()` будет выброшено исключение о несовпадении типов).

**Ресурсы:** [API: ExpressionVisitor.VisitMethodCall](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expressionvisitor.visitmethodcall)

---

### 52. [A] Как посчитать сумму в дереве вида `1 + (2 + (3 + 4))` через визитор, «спускающийся» до листьев?

**Ответ.** Официальный пример показывает, что аккумулирующий визитор обходит дерево в глубину (depth-first): чтобы посчитать значение `BinaryExpression(Add)`, он сначала рекурсивно вычисляет `Left`, потом `Right`, и лишь затем складывает результаты — то есть вычисление происходит «снизу вверх» относительно порядка вызовов, хотя сам обход стартует «сверху». Важное наблюдение из документации: **порядок скобок не хранится в дереве** — `(1 + a) + (3 + b)` и `1 + a + 3 + 4` дают структурно разные деревья (левоассоциативные для операторов без явных скобок), но после построения самих скобок в дереве уже нет — структура узлов сама однозначно определяет порядок вычислений.

**Ресурсы:** [MS Docs: Translate expression trees — Traverse and execute an addition](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-translating#traverse-and-execute-an-addition) · [MS Docs: Interpret expressions](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-interpreting#addition-expression-with-more-operands)

---

### 53. [G] Как реализовать визитор, который «раскрывает» (inlines) вызовы `InvocationExpression`, заменяя их подстановкой параметров (техника, используемая в LINQKit)?

**Ответ.** Идея — при встрече `InvocationExpression`, где `Expression` — это `LambdaExpression`, заменить весь узел на тело этой лямбды, предварительно подставив в него фактические аргументы вызова вместо формальных параметров (комбинация вопроса 50 и `VisitInvocation`):
```csharp
protected override Expression VisitInvocation(InvocationExpression node)
{
    if (node.Expression is LambdaExpression lambda)
    {
        var body = lambda.Body;
        for (int i = 0; i < lambda.Parameters.Count; i++)
            body = new ReplaceParameterVisitor(lambda.Parameters[i], node.Arguments[i]).Visit(body);
        return Visit(body);
    }
    return base.VisitInvocation(node);
}
```
Это то, что решает распространённую ошибку `NotSupportedException: The LINQ expression could not be translated` при использовании динамически скомбинированных предикатов в EF-запросах: без «разворачивания» `InvocationExpression` перед выполнением, провайдер натыкается на узел, который не умеет транслировать в SQL.

**Ресурсы:** [LINQKit — исходный код Expand](https://github.com/scottksmith95/LINQKit) · [API: InvocationExpression](https://learn.microsoft.com/dotnet/api/system.linq.expressions.invocationexpression)

---

### 54. [I] Чем `VisitLambda<T>` отличается от других `VisitXxx`, ведь `LambdaExpression` уже обобщённый тип?

**Ответ.** `ExpressionVisitor.VisitLambda<T>(Expression<T> node)` — обобщённый метод (в отличие от, например, `VisitBinary(BinaryExpression node)`), потому что сама `Expression<TDelegate>` — generic-класс, и переопределение должно сохранять тип делегата на выходе (`Expression<T>`, а не просто `LambdaExpression`). Реализация по умолчанию посещает `Body` и все `Parameters`, и если что-либо изменилось — вызывает `Expression.Lambda<T>(newBody, newParameters)`, сохраняя точный тип `T`. Это гарантирует, что после трансформации `Expression<Func<T,bool>>` останется именно `Expression<Func<T,bool>>`, а не превратится в нетипизированный `LambdaExpression`.

**Ресурсы:** [API: ExpressionVisitor.VisitLambda](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expressionvisitor.visitlambda)

---

### 55. [A] Как написать визитор, собирающий список всех используемых в дереве свойств/полей (`MemberExpression`) — например, для анализа зависимостей проекции?

**Ответ.**
```csharp
public class MemberCollector : ExpressionVisitor
{
    public HashSet<MemberInfo> Members { get; } = new();

    protected override Expression VisitMember(MemberExpression node)
    {
        Members.Add(node.Member);
        return base.VisitMember(node);
    }
}

var collector = new MemberCollector();
collector.Visit(someExpression);
// collector.Members теперь содержит все PropertyInfo/FieldInfo, к которым было обращение
```
Такой визитор используется, например, чтобы понять, какие свойства сущности реально читаются в проекции (`Select`), и на основе этого сгенерировать минимальный SQL `SELECT` — именно эту задачу решает `MemberInitExpression.Bindings`-анализ внутри провайдеров EF/AutoMapper `ProjectTo`.

**Ресурсы:** [API: ExpressionVisitor.VisitMember](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expressionvisitor.visitmember) · [AutoMapper (NuGet)](https://www.nuget.org/packages/AutoMapper)

---

### 56. [G] Почему `ExpressionVisitor` возвращает тот же самый объект узла (по ссылке), если ничего не изменилось, и почему это важно для производительности?

**Ответ.** Базовые реализации `VisitXxx` в `ExpressionVisitor` явно проверяют: если после рекурсивного посещения всех дочерних выражений ни одно из них не изменилось (совпадает по ссылке с исходным), метод возвращает **исходный** узел `node`, а не создаёт новый идентичный объект. Это принципиально для больших деревьев (например, всего LINQ-запроса EF Core с десятками `Where`/`Join`/`Select`): если трансформация затрагивает только один узел глубоко внутри дерева, все «непострадавшие» ветки переиспользуются без аллокаций, а не копируются целиком. Это также упрощает последующее сравнение «изменилось ли дерево» простой проверкой `ReferenceEquals(originalRoot, visitedRoot)`.

**Ресурсы:** [API: ExpressionVisitor](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expressionvisitor) · [MS Docs: Translate expression trees](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-translating)

---

### 57. [A] Как обойти дерево, если нужно посетить узлы, для которых `ExpressionVisitor` не предоставляет специального `VisitXxx` (редкие/новые узлы), не переопределяя весь класс?

**Ответ.** Все специализированные `VisitXxx` в конечном счёте вызываются диспетчером `Visit(Expression node)`, который определяет реальный тип по `node.NodeType`/динамическому типу и делегирует. Если переопределить сам `Visit(Expression node)`, можно перехватывать **все** узлы централизованно (например, для логирования/трассировки каждого посещаемого узла), вызывая `base.Visit(node)` для продолжения стандартной маршрутизации:
```csharp
protected override Expression Visit(Expression node)
{
    if (node != null) Trace.WriteLine($"{node.NodeType}: {node.Type}");
    return base.Visit(node);
}
```
Это работает для всех типов узлов, включая специфичные для конкретной версии .NET/провайдера, без необходимости знать заранее полный список типов.

**Ресурсы:** [API: ExpressionVisitor.Visit](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expressionvisitor.visit)

---
## Группа 6: Компиляция: `Compile()` изнутри

### 58. [I] Что делает `LambdaExpression.Compile()` "под капотом"?

**Ответ.** `Compile()` обходит дерево и генерирует **IL-код** во время выполнения через инфраструктуру, аналогичную `System.Reflection.Emit` (внутри CLR это выделенный `LambdaCompiler`, использующий `DynamicMethod`), затем создаёт из этого IL исполняемый делегат нужного типа (`TDelegate`) и привязывает его к сгенерированному методу. Результат — обычный .NET-делегат, который JIT скомпилирует в машинный код при первом вызове, как любой другой метод. То есть «интерпретации» на лету при каждом вызове не происходит — `Compile()` платит цену один раз (генерация IL + JIT), а дальше делегат выполняется настолько же быстро, как написанный вручную эквивалентный код (с поправкой на то, что сгенерированный код иногда чуть менее оптимален, чем написанный компилятором C# напрямую).

**Диаграмма пайплайна:**
```
Expression Tree  →  LambdaCompiler (обход дерева)  →  IL (через DynamicMethod)
                                                              │
                                                              ▼
                                                     JIT-компиляция при 1-м вызове
                                                              │
                                                              ▼
                                                        Делегат TDelegate
```

**Ресурсы:** [MS Docs: Execute expression trees](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-execution) · [API: LambdaExpression.Compile](https://learn.microsoft.com/dotnet/api/system.linq.expressions.lambdaexpression.compile)

---

### 59. [I] Чем отличаются `Compile()` и `CompileToMethod()`?

**Ответ.** `Compile()` возвращает делегат, готовый к немедленному вызову — компиляция происходит «на лету» через `DynamicMethod`, привязанный к временной сборке в памяти (без возможности сохранить результат на диск). `CompileToMethod(MethodBuilder, DebugInfoGenerator)` записывает сгенерированный IL в **заранее подготовленный** `MethodBuilder` внутри динамически создаваемого модуля/сборки (`AssemblyBuilder`/`ModuleBuilder`), что позволяло, например, сохранить skeleton-сборку на диск через `AssemblyBuilder.Save` (в старом .NET Framework) для последующего использования без повторной генерации кода при следующем запуске. **Важно**: `CompileToMethod` доступен **только в .NET Framework** — в .NET Core/.NET 5+ он отсутствует (сохранение сборок на диск через `Reflection.Emit` было убрано из платформы).

**Ресурсы:** [MS Docs: Execute expression trees — Lambda expressions to functions](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-execution#lambda-expressions-to-functions) · [API: LambdaExpression.CompileToMethod](https://learn.microsoft.com/dotnet/api/system.linq.expressions.lambdaexpression.compiletomethod)

---

### 60. [I] Что вернёт вызов делегата после `Compile()` — новый объект каждый раз или можно переиспользовать?

**Ответ.** Делегат, возвращённый `Compile()`, — обычный объект-делегат, который можно сохранить и **переиспользовать многократно** без повторной компиляции; сама операция `Compile()` создаёт новый метод (и, как следствие, потребляет память и CPU-время на генерацию IL), поэтому официальная рекомендация MS — компилировать **один раз** и кэшировать делегат (например, в `static readonly` поле), а не вызывать `Compile()` при каждом использовании выражения.

**Ресурсы:** [MS Docs: Execute expression trees — Execution and lifetimes](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-execution#execution-and-lifetimes)

---

### 61. [A] Официальное предостережение MS: почему **не стоит** писать собственный сложный механизм кэширования по структурному сравнению деревьев перед каждым `Compile()`?

**Ответ.** Документация Microsoft прямо предупреждает: сравнение двух произвольных expression trees на «представляют ли они один и тот же алгоритм» — операция затратная по времени (нужно рекурсивно и семантически сравнивать структуру, а не просто ссылки), и во многих случаях время, потраченное на такое сравнение ради избежания «лишнего» `Compile()`, превышает выгоду от экономии на самой компиляции. Правильная стратегия — кэшировать компилированный делегат по **явному, дешёвому ключу** (например, по сигнатуре запроса/строковому ключу фильтра), заданному на уровне приложения, а не пытаться делать общий структурный diff произвольных деревьев в рантайме.

**Ресурсы:** [MS Docs: Execute expression trees — Caution](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-execution#execution-and-lifetimes)

---

### 62. [A] Что произойдёт, если выражение ссылается на переменную (замыкание), а объект, к которому она относится, реализует `IDisposable` и уже освобождён к моменту вызова делегата?

**Ответ.** Классический MS-пример: делегат, полученный из `Compile()`, замыкает ссылку на объект (например, `Resource : IDisposable`) — если этот объект был создан в `using`-блоке и `Dispose()` уже вызван к моменту фактического вызова скомпилированного делегата, при обращении к его члену внутри тела выражения будет выброшено `ObjectDisposedException` — **в рантайме**, хотя формально это ошибка про времени компиляции/области видимости переменной. Компилятор гарантирует только то, что переменная *существует* в области видимости на момент компиляции дерева — но не то, что объект, на который она указывает, останется валидным к моменту вызова. Рекомендация: быть осторожным с захватом локальных переменных, ссылающихся на `IDisposable`-ресурсы, особенно при возврате скомпилированного делегата из метода через публичный API.

**Ресурсы:** [MS Docs: Execute expression trees — Caveats](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-execution#caveats)

---

### 63. [I] Что произойдёт, если тело выражения ссылается на метод/тип из сборки, которая недоступна во время `Compile()`?

**Ответ.** Будет выброшено исключение (в частности, для нетипизированного `MethodInfo`/`Type`, найденных, но принадлежащих недогруженной или выгруженной сборке) — документация упоминает `ReferencedAssemblyNotFoundException`-подобные сценарии: сборка, содержащая метод/тип, использованный в дереве, должна быть доступна на всех трёх этапах — при определении выражения (когда получают `MethodInfo`/`PropertyInfo` через рефлексию), при компиляции (`Compile()`) и при вызове результирующего делегата.

**Ресурсы:** [MS Docs: Execute expression trees — Caveats](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-execution#caveats)

---

### 64. [I] Что делать, если тип делегата дерева заранее неизвестен (`LambdaExpression`, а не `Expression<TDelegate>`)?

**Ответ.** Для `LambdaExpression` компилятор всё равно предоставляет `Compile()`, возвращающий обычный `Delegate` (базовый тип, а не конкретный `TDelegate`). Поскольку конкретный тип неизвестен на этапе компиляции C#-кода, вызвать его напрямую через оператор `()` нельзя — нужно использовать `Delegate.DynamicInvoke(object[] args)`, который вызывает делегат через рефлексию, упаковывая/распаковывая аргументы. Это заметно медленнее прямого вызова типизированного делегата (упаковка value-типов в `object`, дополнительная проверка типов аргументов на каждый вызов) — поэтому там, где тип известен заранее, стоит использовать типизированный `Expression<TDelegate>`.

**Ресурсы:** [MS Docs: Execute expression trees — Note](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-execution) · [API: Delegate.DynamicInvoke](https://learn.microsoft.com/dotnet/api/system.delegate.dynamicinvoke)

---

### 65. [A] Насколько дорога операция `Compile()` по сравнению с вызовом уже скомпилированного делегата, и почему это важно в горячих путях кода?

**Ответ.** `Compile()` — относительно тяжёлая операция: она требует обхода всего дерева, генерации IL-инструкций через `Reflection.Emit`-подобный механизм и создания `DynamicMethod`. Это на порядки медленнее одного вызова уже готового делегата (микросекунды против наносекунд, порядок величин зависит от сложности дерева). Классическая ошибка производительности — вызывать `expr.Compile()` **внутри цикла** или на каждый HTTP-запрос вместо того, чтобы скомпилировать один раз при старте/первом использовании и закэшировать делегат (например, в `ConcurrentDictionary<TKey, Delegate>` или `static readonly` поле, если выражение статическое).

**Ресурсы:** [MS Docs: Execute expression trees — Execution and lifetimes](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-execution#execution-and-lifetimes)

---

### 66. [G] Как можно скомпилировать дерево «интерпретируемо» (без генерации IL), и когда это может быть предпочтительнее?

**Ответ.** В .NET (начиная с определённых версий CLR, где `Compile()` внутри способен выбирать между IL-компиляцией и интерпретацией через `Expression.Compile(preferInterpretation: true)`) существует режим «интерпретации» дерева вместо генерации IL — узлы обходятся напрямую специальным интерпретатором на каждом вызове, без затрат на JIT. Это может быть выгодно, когда дерево используется **один-два раза** (цена компиляции IL и последующего JIT превышает выгоду), либо в средах, где генерация кода недоступна/ограничена (например, некоторые AOT/ограниченные окружения, где `Reflection.Emit` запрещён или дорог). Цена — заметно более медленное *исполнение* каждого отдельного вызова по сравнению со скомпилированным в IL и JIT-нативный код делегатом.

**Ресурсы:** [API: Expression.Compile(bool preferInterpretation)](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expression-1.compile)

---

### 67. [I] Как измерить/сравнить накладные расходы: прямой вызов метода vs делегат из `Compile()` vs `DynamicInvoke` vs рефлексия (`MethodInfo.Invoke`)?

**Ответ.** По убыванию скорости выполнения (типичное соотношение, порядок величин зависит от сценария и версии рантайма):
1. **Прямой вызов метода / скомпилированный `Expression.Compile()` делегат** — почти идентичная скорость после JIT (в обоих случаях выполняется нативный машинный код).
2. **`MethodInfo.Invoke`** — заметно медленнее (упаковка аргументов, проверки безопасности/доступа, боксинг value-типов), но с оговоркой, что современный рантайм существенно оптимизировал этот путь по сравнению со старым .NET Framework.
3. **`Delegate.DynamicInvoke`** — обычно самый медленный вариант из типичных: сочетает накладные расходы динамической диспетчеризации и упаковки аргументов в `object[]`.

Практический вывод: для горячих путей, где сигнатура известна заранее, стоит компилировать в типизированный `Expression<TDelegate>` и вызывать напрямую, а не через `DynamicInvoke`/`MethodInfo.Invoke`. Для точных цифр в конкретном проекте всегда рекомендуется профилировать через BenchmarkDotNet, а не полагаться на общие ориентиры.

**Ресурсы:** [API: MethodInfo.Invoke](https://learn.microsoft.com/dotnet/api/system.reflection.methodbase.invoke) · [API: Delegate.DynamicInvoke](https://learn.microsoft.com/dotnet/api/system.delegate.dynamicinvoke)

---
## Группа 7: LINQ to Objects vs LINQ to Queryable

### 68. [I] В чём фундаментальная разница между `IEnumerable<T>` и `IQueryable<T>` с точки зрения expression trees?

**Ответ.** `IEnumerable<T>`-методы расширения (`System.Linq.Enumerable`) принимают **делегаты** (`Func<T,bool>`) и выполняются немедленно/лениво **в памяти** — лямбда компилируется в обычный IL, обхода дерева нет вообще. `IQueryable<T>`-методы расширения (`System.Linq.Queryable`) принимают **`Expression<TDelegate>`** и не выполняют код напрямую: они лишь **накапливают** expression tree, представляющую весь запрос целиком, откладывая выполнение до момента перечисления (`foreach`, `.ToList()`) или вызова `IQueryProvider.Execute`. Именно на этом этапе конкретный провайдер (EF Core, LINQ to SQL, тестовый in-memory провайдер) решает, как перевести накопленное дерево в реальные операции — SQL-запрос, HTTP-вызов, что угодно.

**Диаграмма.**
```
IEnumerable<T>.Where(Func<T,bool>)         →  делегат вызывается сразу, в CLR, построчно
IQueryable<T>.Where(Expression<Func<T,bool>>) →  узел добавляется в дерево
                                                    │
                                            .ToList()/foreach
                                                    │
                                                    ▼
                                       IQueryProvider.Execute(expr)
                                                    │
                                                    ▼
                                         SQL / HTTP / другое исполнение
```

**Ресурсы:** [MS Docs: Introduction to LINQ Queries — The Data Source](https://learn.microsoft.com/dotnet/csharp/linq/get-started/introduction-to-linq-queries#the-data-source) · [API: IQueryable<T>](https://learn.microsoft.com/dotnet/api/system.linq.iqueryable-1)

---

### 69. [I] Что такое `IQueryProvider` и какие два его метода ключевые?

**Ответ.** `IQueryProvider` — интерфейс, который реализует конкретный поставщик данных (например, `DbContext` в EF Core через внутренний провайдер). Два ключевых метода:
- `CreateQuery<TElement>(Expression)` / `CreateQuery(Expression)` — принимает дерево и возвращает новый `IQueryable<TElement>`, «оборачивающий» это дерево (используется при построении цепочки — каждый `.Where()`/`.Select()` вызывает `CreateQuery` с новым, более полным деревом).
- `Execute<TResult>(Expression)` / `Execute(Expression)` — фактически **выполняет** запрос, представленный деревом, и возвращает результат (используется, когда запрос возвращает единичное значение — `.Count()`, `.First()`, — либо когда происходит перечисление `IQueryable`, приводящее к материализации).

**Ресурсы:** [API: IQueryProvider](https://learn.microsoft.com/dotnet/api/system.linq.iqueryprovider) · [API: IQueryProvider.Execute](https://learn.microsoft.com/dotnet/api/system.linq.iqueryprovider.execute)

---

### 70. [I] Что происходит «под капотом» при вызове `dbSet.Where(x => x.Age > 18)`?

**Ответ.** `Where` — статический метод расширения `Queryable.Where<T>(IQueryable<T>, Expression<Func<T,bool>>)`. Компилятор строит `Expression<Func<T,bool>>` из лямбды, а сам вызов `Where` внутри себя формирует новый узел `MethodCallExpression`, где:
- `Method` — `MethodInfo` самого `Queryable.Where` (обобщённого, конкретизированного под `T`);
- `Arguments[0]` — expression, представляющее исходный `IQueryable<T>` (обычно `ConstantExpression`, ссылающийся на `DbSet<T>`, либо вложенный `MethodCallExpression` от предыдущей операции в цепочке);
- `Arguments[1]` — `Quote`-обёрнутая лямбда-предикат.

Затем вызывается `source.Provider.CreateQuery<T>(thisNewMethodCallExpression)`, что и возвращает новый `IQueryable<T>` с накопленным деревом, но **без фактического похода в БД** — до момента перечисления.

**Ресурсы:** [MS Docs: How EF Core queries work](https://learn.microsoft.com/ef/core/querying/how-query-works) · [API: Queryable.Where](https://learn.microsoft.com/dotnet/api/system.linq.queryable.where)

---

### 71. [I] Почему цепочка `.Where().Select().OrderBy()` не выполняется до вызова `.ToList()`/`foreach` (отложенное выполнение / deferred execution)?

**Ответ.** Каждый метод из `Queryable` лишь достраивает дерево и возвращает новый `IQueryable<T>` — реального обращения к данным не происходит. Выполнение запускается только в двух случаях: (1) объект `IQueryable<T>` перечисляется (реализует `IEnumerable<T>`, и `GetEnumerator()` внутри провайдера триггерит выполнение запроса и возврат результатов построчно), либо (2) вызывается терминальный метод, требующий единичного значения (`Count()`, `Sum()`, `First()`, `Any()`), который внутри себя вызывает `provider.Execute<TResult>(expr)`. Это архитектурное решение позволяет провайдеру видеть **весь** запрос целиком перед трансляцией — если бы `Where` выполнялся немедленно, каждый шаг цепочки приходилось бы транслировать и выполнять отдельным запросом к БД, что крайне неэффективно.

**Ресурсы:** [MS Docs: IQueryable<T> Interface — Remarks](https://learn.microsoft.com/dotnet/api/system.linq.iqueryable-1#remarks) · [MS Docs: Write LINQ queries — deferred execution](https://learn.microsoft.com/dotnet/csharp/linq/get-started/write-linq-queries)

---

### 72. [A] Почему вызов `.AsEnumerable()` посреди LINQ-цепочки на `IQueryable<T>` — важная и осознанная операция?

**Ответ.** `.AsEnumerable()` меняет статический тип с `IQueryable<T>` на `IEnumerable<T>`, из-за чего все **последующие** вызовы в цепочке разрешаются в перегрузки `Enumerable` (делегаты, LINQ to Objects), а не `Queryable` (expression trees). Это фактически граница «то, что транслируется провайдером» / «то, что выполняется в памяти после материализации предыдущей части». Часто используется намеренно, когда нужная операция не поддерживается провайдером (например, вызов сложного C#-метода, который EF Core не умеет транслировать в SQL) — тогда часть запроса до `.AsEnumerable()` выполняется на сервере БД, а всё, что после, — на клиенте, в памяти, уже над материализованными объектами. Неосознанное использование, наоборот, — частая причина непредвиденной загрузки в память всей таблицы до применения фильтра.

**Ресурсы:** [API: Queryable.AsEnumerable](https://learn.microsoft.com/dotnet/api/system.linq.queryable.asenumerable)

---

### 73. [I] Что делает `EnumerableQuery<T>` и как связан с `.AsQueryable()` над `List<T>`?

**Ответ.** `.AsQueryable()`, вызванный над обычной коллекцией в памяти (`IEnumerable<T>`), оборачивает её в `EnumerableQuery<T>` — встроенную реализацию `IQueryable<T>`, чей `IQueryProvider` **не переводит дерево в другой язык запросов**, а просто компилирует накопленное дерево через `Expression.Compile()` и выполняет его как обычный LINQ to Objects запрос при перечислении. Это удобно для написания кода, единообразно работающего и с `IQueryable`, и с `IEnumerable` (например, в тестах — подменить `DbSet<T>` на `List<T>.AsQueryable()`), но нужно помнить: `EnumerableQuery` не проверяет, «переводим» ли запрос в SQL — она просто скомпилирует и выполнит **любое** валидное C#-выражение, из-за чего юнит-тесты на in-memory коллекции могут «пропускать» ошибки трансляции, которые проявятся только на реальном провайдере (EF Core).

**Ресурсы:** [API: EnumerableQuery<T>](https://learn.microsoft.com/dotnet/api/system.linq.enumerablequery-1) · [API: Queryable.AsQueryable](https://learn.microsoft.com/dotnet/api/system.linq.queryable.asqueryable)

---

### 74. [A] Почему написание собственного `IQueryable`-провайдера — сложная задача, и какова классическая точка входа для изучения этой темы?

**Ответ.** Сложность в том, что провайдер должен обработать **произвольное** дерево, порождаемое сколь угодно сложной комбинацией стандартных LINQ-операторов (`Where`, `Select`, `Join`, `GroupBy`, `OrderBy`, вложенные подзапросы, агрегаты), корректно сохраняя семантику каждого — включая тонкости вроде отложенных `Join`, ленивой загрузки навигационных свойств, различий между `Select` над скалярами и над сложными проекциями. Классический учебный материал — серия блог-постов Мэтта Уоррена (одного из авторов LINQ to SQL) «LINQ: Building an IQueryable Provider», подробно разбирающая построение минимального, но полноценного провайдера с нуля, вплоть до генерации SQL. Современная production-реализация такого уровня сложности — это, по сути, то, чем является ядро EF Core (`RelationalSqlTranslatingExpressionVisitor` и связанные компоненты).

**Ресурсы:** [Matt Warren: LINQ Building an IQueryable Provider — Part I](https://learn.microsoft.com/en-us/archive/blogs/mattwar/linq-building-an-iqueryable-provider-part-i) · [Полный список серии](https://learn.microsoft.com/en-us/archive/blogs/mattwar/linq-building-an-iqueryable-provider-series) · [IQToolkit на GitHub](https://github.com/mattwar/iqtoolkit)

---

### 75. [I] Почему в query-syntax LINQ (`from x in ... where ... select ...`) не рекомендуется использовать паттерн-матчинг (`is null`, `is not null`)?

**Ответ.** Официальная документация прямо предупреждает: хотя современный C# позволяет писать `x is null` вместо `x == null`, паттерн-матчинг — относительно новая синтаксическая конструкция, и не все LINQ-провайдеры (реализации `IQueryProvider`, транслирующие дерево в SQL/другой язык) гарантированно умеют корректно интерпретировать порождаемые ей узлы дерева. Рекомендуется использовать `equals`/`==`/`!=` в LINQ-запросах, ориентированных на `IQueryable`-источники, чтобы не рисковать неожиданным поведением или исключением трансляции на менее актуальных версиях провайдера.

**Ресурсы:** [MS Docs: Write C# LINQ queries — Handle null values](https://learn.microsoft.com/dotnet/csharp/linq/get-started/write-linq-queries#handle-null-values-in-query-expressions)

---

### 76. [A] Как `IQueryable<T>.Expression` помогает при отладке сложного LINQ-запроса до его выполнения?

**Ответ.** Свойство `IQueryable.Expression` возвращает полностью накопленное дерево запроса **до** его выполнения — это позволяет, например, вывести `queryable.Expression.ToString()` (или использовать сторонние библиотеки визуализации дерева, см. группу про отладку) в отладчике/логе, чтобы увидеть точную структуру запроса (все вложенные `MethodCallExpression` от `Where`/`Select`/`OrderBy`), не дожидаясь материализации и не читая финальный SQL. EF Core дополнительно предоставляет собственные механизмы логирования сгенерированного SQL (`ILogger`/`.ToQueryString()`), но `Expression`-свойство — универсальный, провайдеро-независимый способ увидеть «что вообще было построено».

**Ресурсы:** [API: IQueryable.Expression](https://learn.microsoft.com/dotnet/api/system.linq.iqueryable.expression) · [EF Core: ToQueryString](https://learn.microsoft.com/ef/core/querying/how-query-works)

---

### 77. [I] Почему смешивание в одном запросе клиентских (client-side) методов и провайдер-специфичных операций иногда «работает», а иногда бросает исключение трансляции?

**Ответ.** Современные версии EF Core (начиная с EF Core 3.0) явно запрещают «молчаливое» разбиение запроса — вычисление части выражения на клиенте без явного разделения через `.AsEnumerable()`/`.ToList()` — за пределами самого верхнего уровня проекции (`Select`); при попытке использовать неподдерживаемый метод внутри `Where`/`Join`/сложной части запроса провайдер бросает `InvalidOperationException` («could not be translated»), а не тихо переключается на клиентское выполнение (как это упрощённо делали более ранние версии EF6/LINQ to SQL для некоторых сценариев). Это осознанное архитектурное решение — предсказуемость важнее «магии», из-за которой раньше можно было случайно перекачать всю таблицу в память, не заметив этого.

**Ресурсы:** [EF Core: Client vs. Server Evaluation](https://learn.microsoft.com/ef/core/querying/client-eval) · [What's New in EF Core 3.0 — query pipeline](https://learn.microsoft.com/ef/core/what-is-new/ef-core-3.0/breaking-changes)

---
## Группа 8: EF Core и провайдеры: трансляция в SQL

### 78. [A] На верхнем уровне: какие этапы проходит LINQ-запрос EF Core от `Expression Tree` до SQL?

**Ответ.** Упрощённый конвейер (детали см. в документации «How query processing works»):
1. **Парсинг дерева запроса** — EF Core получает `Expression` от `IQueryable`-цепочки.
2. **Query compilation pipeline**: серия `ExpressionVisitor`-проходов — навигационные свойства «разворачиваются» в join-структуры, применяются правила включения (`Include`), обрабатывается client vs. server evaluation.
3. **`RelationalSqlTranslatingExpressionVisitor`** — специализированный визитор, транслирующий C#-узлы (`BinaryExpression`, `MethodCallExpression` и т.д.) в собственное промежуточное SQL-дерево-представление (`SqlExpression` и наследники).
4. **Генерация SQL-текста** из промежуточного SQL-дерева конкретным диалектом (SQL Server/PostgreSQL/SQLite и т.д.).
5. **Кэширование скомпилированного плана запроса** (query plan cache) по структурному ключу дерева — чтобы для одинаковой по форме, но разной по параметрам последовательности вызовов не пересобирать SQL каждый раз.
6. Выполнение через `DbCommand`, материализация результатов обратно в объекты домена.

**Диаграмма.**
```
IQueryable<T> (Expression Tree, C#-семантика)
        │  ExpressionVisitor pass'ы EF Core (navigation expansion, Include...)
        ▼
Внутреннее QueryModel / ShapedQueryExpression
        │  RelationalSqlTranslatingExpressionVisitor
        ▼
SqlExpression-дерево (SELECT/WHERE/JOIN как узлы)
        │  QuerySqlGenerator конкретного провайдера (SQL Server/PostgreSQL...)
        ▼
Текст SQL + параметры  →  DbCommand.ExecuteReader  →  материализация в объекты
```

**Ресурсы:** [EF Core: How query processing works](https://learn.microsoft.com/ef/core/querying/how-query-works) · [RelationalSqlTranslatingExpressionVisitor](https://learn.microsoft.com/dotnet/api/microsoft.entityframeworkcore.query.relationalsqltranslatingexpressionvisitor)

---

### 79. [I] Почему EF Core обычно параметризует значения переменных в SQL, а константы из выражения встраивает буквально?

**Ответ.** Например, `context.Blogs.Where(b => b.Id == id)`, где `id` — локальная переменная (захваченная в замыкании как `MemberExpression` на поле closure-класса, см. группу про замыкания), транслируется как параметризованный SQL: `WHERE [b].[Id] = @__id_0`. Это сделано намеренно ради переиспользования плана запроса SQL-сервером (один и тот же скомпилированный план для разных значений `id`) и защиты от SQL-инъекций. Буквальные константы прямо в C#-выражении (например, `b.Id == 5`, где `5` — `ConstantExpression`, известная на этапе компиляции LINQ-запроса) в некоторых случаях (не всегда) могут встраиваться как литерал в текст SQL — начиная с EF Core 9 появилась возможность явно управлять этим поведением через `EF.Constant`/`EF.Parameter`.

**Ресурсы:** [What's New in EF Core 9 — force/prevent parameterization](https://learn.microsoft.com/ef/core/what-is-new/ef-core-9.0/whatsnew#linq-and-sql-translation)

---

### 80. [I] Почему запрос `context.Customers.Where(c => c.City == "London")` работает, а `context.Customers.Where(c => SomeLocalHelperMethod(c))` может выбросить `InvalidOperationException`?

**Ответ.** EF Core транслирует в SQL только те конструкции дерева, для которых существует зарегистрированное правило перевода (например, стандартные операторы сравнения, часть методов `string`/`DateTime`/`Math`, поддерживаемые провайдером). Произвольный пользовательский метод (`SomeLocalHelperMethod`) — это `MethodCallExpression`, ссылающийся на C#-метод, у которого **нет** SQL-эквивалента и который EF Core не может «заглянуть внутрь» и развернуть (если только это не выражение, зарегистрированное явно как функция БД через `[DbFunction]`/`ModelBuilder.HasDbFunction`, либо не помечено для клиентского вычисления). Решение: либо переписать условие через поддерживаемые конструкции, либо явно материализовать данные раньше (`.AsEnumerable()`/`.ToList()`) и применить метод уже в памяти.

**Ресурсы:** [EF Core: Client vs. Server Evaluation](https://learn.microsoft.com/ef/core/querying/client-eval) · [EF Core: User-defined functions](https://learn.microsoft.com/ef/core/querying/user-defined-function-mapping)

---

### 81. [A] Как EF Core транслирует `Select(x => new Dto { Id = x.Id, Name = x.Name })` в список выбираемых колонок, а не `SELECT *`?

**Ответ.** Аргумент `Select` — `Expression<Func<T,TDto>>`, тело которого (после разворачивания `Quote`) обычно представляет `MemberInitExpression`: `NewExpression` (вызов конструктора `Dto`) плюс список `Bindings` (`MemberAssignment` для каждого свойства). `RelationalSqlTranslatingExpressionVisitor` рекурсивно обходит каждый `MemberAssignment.Expression` (в данном случае — `MemberExpression` `x.Id` и `x.Name`), транслируя каждое отдельно в соответствующее SQL-выражение колонки, и собирает итоговый список выбираемых столбцов именно из этого набора — колонки, не упомянутые ни в одном `Binding`, физически не попадают в текст `SELECT`. Это и есть техническая причина, почему проекции в EF Core эффективнее, чем загрузка полной сущности с последующим маппингом в памяти.

**Ресурсы:** [EF Core: Projection queries](https://learn.microsoft.com/ef/core/querying/select) · [API: MemberInitExpression](https://learn.microsoft.com/dotnet/api/system.linq.expressions.memberinitexpression)

---

### 82. [A] Как `.Include(x => x.Orders)` использует expression tree, если он вообще не описывает условие/значение, а «путь» навигации?

**Ответ.** `Include<TEntity, TProperty>(IQueryable<TEntity>, Expression<Func<TEntity, TProperty>>)` принимает лямбду не для трансляции в предикат, а как **путь доступа к навигационному свойству** — EF Core разбирает тело выражения (обычно простой `MemberExpression`, например `x => x.Orders`), извлекает имя свойства через `((MemberExpression)expr.Body).Member`, и на основе этого настраивает eager loading соответствующей навигации в модели метаданных. Здесь expression tree используется не для генерации SQL-условия напрямую, а как типобезопасный, «rename-friendly» аналог передачи строкового имени свойства (`.Include("Orders")` — старый, менее безопасный вариант API, всё ещё поддерживаемый для динамических сценариев).

**Ресурсы:** [EF Core: Loading related data — Include](https://learn.microsoft.com/ef/core/querying/related-data/eager) · [API: EntityFrameworkQueryableExtensions.Include](https://learn.microsoft.com/dotnet/api/microsoft.entityframeworkcore.entityframeworkqueryableextensions.include)

---

### 83. [I] Что означает предупреждение «query could not be translated and will be evaluated locally» / ошибка трансляции, и как её отладить?

**Ответ.** Это сигнал, что конкретный узел дерева запроса не имеет зарегистрированного правила перевода у используемого провайдера. Порядок отладки: 1) прочитать полное сообщение исключения — оно обычно указывает конкретную часть LINQ-выражения, вызвавшую проблему; 2) включить логирование сгенерированного SQL/чувствительных данных (`UseLoggerFactory`, `EnableSensitiveDataLogging`) для контекста; 3) попробовать вызвать `.ToQueryString()` на `IQueryable`, чтобы увидеть, до какого места запрос вообще транслируется; 4) переписать неподдерживаемую часть через поддерживаемые конструкции или `[DbFunction]`; 5) в крайнем случае — явно разбить запрос на серверную и клиентскую часть через `.AsEnumerable()`, осознавая последствия для производительности (загрузка большего объёма данных).

**Ресурсы:** [EF Core: How query processing works](https://learn.microsoft.com/ef/core/querying/how-query-works) · [EF Core: Client vs. Server Evaluation](https://learn.microsoft.com/ef/core/querying/client-eval)

---

### 84. [A] Как эволюционировала трансляция LINQ в SQL между версиями EF Core — приведите конкретный пример улучшения.

**Ответ.** Каждая мажорная версия EF Core расширяет набор поддерживаемых трансляций. Примеры из документации:
- EF Core 6: перевод `String.Concat` с множественными аргументами в SQL-конкатенацию.
- EF Core 7: перевод `String.IndexOf`, перевод `Object.GetType()` для проверки типа в TPH-иерархиях, трансляции статистических агрегатных функций.
- EF Core 9: перевод `Math.Min`/`Math.Max` через SQL `GREATEST`/`LEAST` (для СУБД, поддерживающих эти функции), возможность форсировать/предотвратить параметризацию через `EF.Constant`/`EF.Parameter`, оптимизация подзапросов `COUNT(*)` без лишней проекции.
- EF Core 10: более консистентный порядок в split queries, перевод `DateOnly.ToDateTime()`/`DateOnly.DayNumber`, по умолчанию — множественные параметры вместо JSON-массива для `Contains` над коллекциями.

Это иллюстрирует, что «список поддерживаемых конструкций» — не статичен, и один и тот же LINQ-запрос со временем может начать транслироваться в более эффективный (или просто *начать* транслироваться, если раньше не поддерживался) SQL при обновлении версии EF Core.

**Ресурсы:** [What's New in EF Core 6.0](https://learn.microsoft.com/ef/core/what-is-new/ef-core-6.0/whatsnew#linq-query-enhancements) · [What's New in EF Core 7.0](https://learn.microsoft.com/ef/core/what-is-new/ef-core-7.0/whatsnew#query-enhancements) · [What's New in EF Core 9](https://learn.microsoft.com/ef/core/what-is-new/ef-core-9.0/whatsnew#linq-and-sql-translation) · [What's New in EF Core 10](https://learn.microsoft.com/ef/core/what-is-new/ef-core-10.0/whatsnew#linq-and-sql-translation)

---

### 85. [G] Зачем EF Core внутри использует собственный `ReplacingExpressionVisitor`, а не общий `ExpressionVisitor` напрямую в каждом месте?

**Ответ.** `ReplacingExpressionVisitor` (внутренний тип EF Core, `Microsoft.EntityFrameworkCore.Query`) — специализированная реализация `ExpressionVisitor`, заточенная под одну конкретную, часто повторяющуюся задачу: заменить одно конкретное подвыражение на другое во всём дереве (по сути — обобщение паттерна из вопроса 50 про подстановку параметров, но для произвольных пар «искомое/замена», а не только `ParameterExpression`). Это используется, например, при разворачивании навигационных свойств, инлайнинге определений `[DbFunction]`, объединении под-запросов. Наличие такого готового, хорошо протестированного визитора в публичном API (`Microsoft.EntityFrameworkCore.Query.ReplacingExpressionVisitor`) полезно и авторам сторонних провайдеров/расширений EF Core, которым не нужно писать этот код заново.

**Ресурсы:** [API: ReplacingExpressionVisitor](https://learn.microsoft.com/dotnet/api/microsoft.entityframeworkcore.query.replacingexpressionvisitor) · [EF Core: Implementation of database providers](https://learn.microsoft.com/ef/core/providers/writing-a-provider)

---

### 86. [I] Как ORM-фреймворк отличает выражение-предикат для `Where` от выражения-проекции для `Select` на уровне expression tree?

**Ответ.** Отличие не в самом узле, а в **позиции** аргумента в `MethodCallExpression` конкретного стандартного оператора запроса: провайдер знает (по сигнатуре `Queryable.Where`/`Queryable.Select`), что второй аргумент `Where` — это `Expression<Func<T,bool>>` (ожидаемый тип возврата — `bool`, интерпретируется как условие фильтрации), а второй аргумент `Select` — `Expression<Func<T,TResult>>` с произвольным `TResult` (интерпретируется как формирователь итогового значения/объекта). То есть семантика определяется не структурой самого выражения, а тем, **в какой стандартный LINQ-оператор** оно было передано — сама библиотека `Expression` не «знает» о понятиях «фильтр» или «проекция», это уже интерпретация со стороны `Queryable`/провайдера.

**Ресурсы:** [API: Queryable.Where](https://learn.microsoft.com/dotnet/api/system.linq.queryable.where) · [API: Queryable.Select](https://learn.microsoft.com/dotnet/api/system.linq.queryable.select)

---

### 87. [A] Что такое кэш скомпилированных запросов (compiled query cache) EF Core и как он связан со структурой дерева?

**Ответ.** Чтобы не проходить весь дорогой конвейер трансляции (парсинг дерева → SQL-генерация) при каждом выполнении «одинакового по форме» запроса (одна и та же цепочка операторов, но разные значения параметров-переменных), EF Core кэширует результат компиляции запроса, используя как ключ **структуру дерева без учёта конкретных значений констант из замыканий** — по сути, «форму» дерева. Если структура (типы узлов, методы, свойства) совпадает с ранее скомпилированным запросом, а различаются только значения параметризованных переменных, EF Core переиспользует уже сгенерированный SQL и план, подставляя новые значения параметров, вместо повторной трансляции с нуля. Отсюда практическая рекомендация — избегать построения LINQ-запросов с **разной структурой** дерева на каждый вызов (например, через динамическую сборку условий в цикле без нормализации), так как это «взрывает» кэш планов запросов.

**Ресурсы:** [EF Core: How query processing works — Query caching](https://learn.microsoft.com/ef/core/querying/how-query-works) · [EF Core: Compiled queries (явные)](https://learn.microsoft.com/ef/core/performance/advanced-performance-topics)

---

### 88. [G] Как Moq использует Expression Trees, чтобы отличить `mock.Setup(x => x.DoWork(It.IsAny<int>()))` от обычного вызова метода?

**Ответ.** Аргумент `Setup` типизирован как `Expression<Action<T>>` (или `Expression<Func<T,TResult>>` для методов с возвращаемым значением) — поэтому переданная лямбда **не выполняется** сразу, а Moq получает `MethodCallExpression`, из которого извлекает: (1) `Method` — какой именно метод настраивается, (2) `Arguments` — какие аргументы ожидаются, при этом специальные конструкции вроде `It.IsAny<int>()` сами являются вызовами методов внутри дерева (`MethodCallExpression`, ссылающийся на `It.IsAny<T>()`), которые Moq распознаёт по сигнатуре и превращает в matcher вместо буквального значения аргумента. Это ключевая причина, почему `Setup` принимает именно `Expression`, а не делегат — вызвать реальный метод на mock-объекте для «конфигурации» было бы бессмысленно (или потребовало бы дополнительного API вроде callback-конфигурации), тогда как дерево позволяет «прочитать намерение», не выполняя код.

**Ресурсы:** [Moq (NuGet)](https://www.nuget.org/packages/Moq) · [API: Expression<TDelegate>](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expression-1)

---

### 89. [A] Как AutoMapper `ProjectTo<TDto>()` использует expression trees для построения эффективной проекции на уровне БД?

**Ответ.** `ProjectTo<TDto>()`, в отличие от `Map<TDto>()` (который сначала материализует полные сущности, а потом маппит их в памяти), строит **expression tree проекции** на основе конфигурации карт (`CreateMap<TSource,TDto>()`), собирая единый `Expression<Func<TSource,TDto>>` (обычно через `MemberInitExpression`, аналогично вопросу 81), и применяет его как аргумент `.Select()` к исходному `IQueryable<TSource>`. Благодаря этому вся проекция (включая вложенные объекты и вычисляемые поля, если они выражены через поддерживаемые конструкции) уходит в единый SQL-запрос через провайдера (EF Core), возвращая из БД только реально нужные столбцы — без загрузки всей сущности целиком.

**Ресурсы:** [AutoMapper (NuGet)](https://www.nuget.org/packages/AutoMapper) · [EF Core: Projection queries](https://learn.microsoft.com/ef/core/querying/select)

---
## Группа 9: Динамические предикаты и `PredicateBuilder`

### 90. [I] Как динамически построить предикат `Expression<Func<T,bool>>` из набора условий, заданных пользователем во время выполнения (например, фильтр в UI)?

**Ответ.** Базовый подход: создать один общий `ParameterExpression`, затем для каждого условия построить соответствующий `BinaryExpression`/`MethodCallExpression`, используя этот же параметр, и объединить их через `Expression.AndAlso`/`Expression.OrElse`, после чего обернуть итог в `Expression.Lambda`:
```csharp
var param = Expression.Parameter(typeof(Person), "p");
Expression? body = null;

if (minAge is int min)
{
    var cond = Expression.GreaterThanOrEqual(Expression.Property(param, nameof(Person.Age)), Expression.Constant(min));
    body = body is null ? cond : Expression.AndAlso(body, cond);
}
if (city is not null)
{
    var cond = Expression.Equal(Expression.Property(param, nameof(Person.City)), Expression.Constant(city));
    body = body is null ? cond : Expression.AndAlso(body, cond);
}

body ??= Expression.Constant(true); // если условий нет — предикат "всё"
var predicate = Expression.Lambda<Func<Person,bool>>(body, param);
```
Ключевое требование — **переиспользовать один и тот же `ParameterExpression`** во всех подусловиях, иначе `Expression.Lambda` не свяжет их корректно (см. вопрос 19).

**Ресурсы:** [MS Docs: How to build dynamic queries](https://learn.microsoft.com/dotnet/csharp/linq/how-to-build-dynamic-queries)

---

### 91. [A] Почему нельзя просто взять два готовых `Expression<Func<T,bool>>`, построенных **независимо**, и объединить их `Expression.AndAlso(a.Body, b.Body)`?

**Ответ.** Если `a` и `b` были построены независимо (например, из двух разных лямбда-выражений компилятором C#), у каждого — **свой собственный** `ParameterExpression` (даже если оба называются `"p"` и имеют тип `Person`, это разные объекты в памяти). `Expression.AndAlso(a.Body, b.Body)` формально скомпилируется, но результирующее дерево будет содержать ссылки на **два разных** параметра — и при попытке обернуть это в `Expression.Lambda<Func<Person,bool>>(combined, singleParam)`, где `singleParam` — только один из двух, второй параметр окажется «свободной переменной», не связанной с лямбдой, и будет выброшено `InvalidOperationException` о незакрытой переменной. Решение — сначала унифицировать параметры (см. следующий вопрос).

**Ресурсы:** [API: ParameterExpression](https://learn.microsoft.com/dotnet/api/system.linq.expressions.parameterexpression) · [LINQKit](https://github.com/scottksmith95/LINQKit)

---

### 92. [A] Как именно `PredicateBuilder` из LINQKit решает проблему объединения независимо построенных предикатов?

**Ответ.** `PredicateBuilder.And`/`Or` берёт тело второго выражения и через `ExpressionVisitor`-подстановку (аналог вопроса 50) заменяет **все вхождения параметра второго выражения** на параметр первого, после чего собирает `BinaryExpression`(AndAlso/OrElse) из тела первого и «перепривязанного» тела второго, оборачивая результат в новую `Expression.Lambda` с единственным (первым) параметром:
```csharp
public static Expression<Func<T,bool>> And<T>(this Expression<Func<T,bool>> expr1, Expression<Func<T,bool>> expr2)
{
    var secondBody = ParameterRebinder.ReplaceParameters(expr2, expr1.Parameters[0]);
    return Expression.Lambda<Func<T,bool>>(Expression.AndAlso(expr1.Body, secondBody), expr1.Parameters[0]);
}
```
Именно на этом основан идиоматичный код:
```csharp
var predicate = PredicateBuilder.New<Person>(true);
if (hasMinAge) predicate = predicate.And(p => p.Age >= minAge);
if (hasCity)   predicate = predicate.And(p => p.City == city);
```

**Ресурсы:** [LINQKit — PredicateBuilder.cs (исходники)](https://github.com/scottksmith95/LINQKit/blob/master/src/LinqKit.Core/PredicateBuilder.cs)

---

### 93. [A] Почему при использовании `PredicateBuilder` с EF Core иногда возникает `NotSupportedException`/`could not be translated`, и как это лечится через `.AsExpandable()`?

**Ответ.** Некоторые способы комбинирования выражений в LINQKit (в частности, через `.Invoke()`/оборачивание в `InvocationExpression`, а не через прямую подстановку параметров как в вопросе 92) оставляют в итоговом дереве узел `InvocationExpression`, который многие провайдеры (в т.ч. исторически EF) не умеют транслировать напрямую в SQL. Решение LINQKit — обернуть исходный `IQueryable` через `.AsExpandable()`: это подменяет `IQueryProvider` на прокси, который перед фактическим выполнением запроса пропускает дерево через собственный `ExpressionVisitor` (`ExpressionExpander`), разворачивающий все `InvocationExpression` в чистые деревья без вызовов делегатов (см. вопрос 53) — и только затем передаёт «очищенное» дерево реальному провайдеру EF Core.

**Ресурсы:** [LINQKit README — AsExpandable](https://github.com/scottksmith95/LINQKit/blob/master/README.md) · [Issue про трансляцию](https://github.com/scottksmith95/LINQKit/issues/140)

---

### 94. [I] Как построить предикат «содержит любую из строк в списке» (`WHERE Name IN (...)`) через Expression API?

**Ответ.** Через `MethodCallExpression`, ссылающийся на `Enumerable.Contains`/`List<T>.Contains`, применённый к `ConstantExpression` со списком значений:
```csharp
var names = new List<string> { "Alice", "Bob" };
var param = Expression.Parameter(typeof(Person), "p");
var nameProp = Expression.Property(param, nameof(Person.Name));
var containsMethod = typeof(List<string>).GetMethod(nameof(List<string>.Contains), new[] { typeof(string) })!;
var call = Expression.Call(Expression.Constant(names), containsMethod, nameProp);
var predicate = Expression.Lambda<Func<Person,bool>>(call, param);
```
EF Core транслирует такой `MethodCallExpression` в SQL `IN (...)` (с деталями параметризации, зависящими от версии — см. вопрос про EF Core 10 breaking change с множественными параметрами вместо JSON-массива).

**Ресурсы:** [Breaking changes EF Core 10 — parameterized collections](https://learn.microsoft.com/ef/core/what-is-new/ef-core-10.0/breaking-changes#low-impact-changes) · [API: Expression.Call](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expression.call)

---

### 95. [A] Как реализовать «Specification pattern» (спецификации предметной области), используя `Expression<Func<T,bool>>` как переиспользуемый строительный блок?

**Ответ.** Идея — инкапсулировать бизнес-правило фильтрации в отдельный класс со свойством/методом, возвращающим `Expression<Func<T,bool>>`, который можно и передать в `.Where()` (для трансляции в SQL), и скомпилировать и вызвать напрямую в памяти (для проверки объекта в коде без похода в БД) — именно поэтому спецификация хранит **дерево**, а не делегат:
```csharp
public abstract class Specification<T>
{
    public abstract Expression<Func<T,bool>> ToExpression();
    public bool IsSatisfiedBy(T entity) => ToExpression().Compile()(entity);
}

public class AdultSpecification : Specification<Person>
{
    public override Expression<Func<Person,bool>> ToExpression() => p => p.Age >= 18;
}

// Использование:
var query = db.People.Where(spec.ToExpression());       // транслируется в SQL
bool ok = spec.IsSatisfiedBy(somePersonInMemory);         // выполняется в памяти
```
Комбинирование нескольких спецификаций (`And`/`Or`/`Not`) обычно реализуется поверх той же техники подстановки параметров, что и `PredicateBuilder`.

**Ресурсы:** [MS Docs: Expression Trees Overview](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/) · [LINQKit](https://github.com/scottksmith95/LINQKit)

---

### 96. [I] Как построить сортировку по имени свойства, заданному строкой во время выполнения (`OrderBy("Name")`)?

**Ответ.**
```csharp
static IQueryable<T> OrderByPropertyName<T>(IQueryable<T> source, string propertyName)
{
    var param = Expression.Parameter(typeof(T), "x");
    var property = Expression.PropertyOrField(param, propertyName);
    var keySelector = Expression.Lambda(property, param);

    var method = typeof(Queryable).GetMethods()
        .First(m => m.Name == nameof(Queryable.OrderBy) && m.GetParameters().Length == 2)
        .MakeGenericMethod(typeof(T), property.Type);

    var result = method.Invoke(null, new object[] { source, keySelector })!;
    return (IQueryable<T>)result;
}
```
Здесь важно: тип-параметр ключа сортировки (`property.Type`) неизвестен на этапе компиляции C#, поэтому обобщённый метод `Queryable.OrderBy<TSource,TKey>` конкретизируется через рефлексию (`MakeGenericMethod`), а сам вызов производится через `MethodInfo.Invoke` — это стандартный паттерн для динамических построителей запросов (в промышленном коде для этого чаще используют готовые библиотеки, например System.Linq.Dynamic.Core, а не пишут вручную).

**Ресурсы:** [API: MethodInfo.MakeGenericMethod](https://learn.microsoft.com/dotnet/api/system.reflection.methodinfo.makegenericmethod) · [System.Linq.Dynamic.Core](https://github.com/zzzprojects/System.Linq.Dynamic.Core)

---

### 97. [A] Как безопасно защититься от «инъекции» произвольного C#-кода при построении предикатов на основе пользовательского ввода (например, имени свойства из UI-фильтра)?

**Ответ.** Поскольку построение выполняется через типизированный Expression API (а не через `eval`-подобный парсинг произвольной строки как C#-кода), «инъекция кода» в классическом смысле здесь невозможна — но есть риски другого рода: (1) `ArgumentException`/исключения от `GetProperty`/`PropertyOrField`, если имя свойства не существует — нужна явная валидация по белому списку разрешённых имён свойств, а не слепое доверие вводу; (2) при использовании библиотек типа System.Linq.Dynamic.Core, которые парсят **произвольные строковые выражения** (`"Age > 18 AND City == \"London\""`) в деревья — там уже есть собственный риск, если строка целиком приходит от пользователя без ограничений (потенциальный DoS через сложные выражения, доступ к нежелательным членам через рефлексию) — эти библиотеки предоставляют настройки ограничения доступных типов/методов именно для таких сценариев.

**Ресурсы:** [System.Linq.Dynamic.Core — Security](https://github.com/zzzprojects/System.Linq.Dynamic.Core) · [API: Expression.PropertyOrField](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expression.propertyorfield)

---

### 98. [I] Чем отличается объединение условий через `Expression.AndAlso` от простого написания `predicate1.Compile()(x) && predicate2.Compile()(x)` в цикле?

**Ответ.** Второй вариант компилирует **оба** выражения в отдельные делегаты и вызывает их последовательно в C#-коде — это работает только «в памяти» (после материализации данных) и требует двух отдельных вызовов `Compile()` (с их стоимостью, см. вопрос 65) на каждый элемент, если не закэшировать делегаты заранее. Первый вариант (`Expression.AndAlso`) объединяет **сами деревья** в одно составное дерево **до** компиляции — такое единое дерево можно передать `IQueryable.Where()`, и оно будет транслировано провайдером в единое SQL-условие `WHERE (...) AND (...)`, что выполнится на сервере БД за один проход, без выкачивания лишних данных в память.

**Ресурсы:** [API: Expression.AndAlso](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expression.andalso)

---

### 99. [G] Как построить предикат, комбинирующий произвольное (динамическое) число условий через `OR`, избегая построения глубоко вложенного (не сбалансированного) дерева при большом количестве условий?

**Ответ.** Наивный `Aggregate` (`conditions.Aggregate(Expression.OrElse)`) строит **левоассоциативное**, линейно нарастающее по глубине дерево — для тысяч условий это может привести к очень глубокой рекурсии при последующем обходе (`Compile()`/визиторы), с риском `StackOverflowException` в патологических случаях. Более устойчивый подход — строить **сбалансированное** дерево (divide-and-conquer: рекурсивно объединять пары условий, а не расширять цепочку линейно), что снижает максимальную глубину рекурсии с O(n) до O(log n):
```csharp
static Expression BuildBalancedOr(IReadOnlyList<Expression> conditions)
{
    if (conditions.Count == 1) return conditions[0];
    int mid = conditions.Count / 2;
    var left = BuildBalancedOr(conditions.Take(mid).ToList());
    var right = BuildBalancedOr(conditions.Skip(mid).ToList());
    return Expression.OrElse(left, right);
}
```
На практике для действительно больших наборов значений предпочтительнее использовать `Contains` над коллекцией (вопрос 94) вместо цепочки `OR` — и провайдер, и сервер БД обрабатывают `IN (...)` эффективнее длинной дизъюнкции.

**Ресурсы:** [API: Expression.OrElse](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expression.orelse)

---
## Группа 10: Dynamic LINQ (`System.Linq.Dynamic.Core`)

### 100. [I] Что решает библиотека `System.Linq.Dynamic.Core` и чем её подход отличается от ручного построения через `Expression` API?

**Ответ.** `System.Linq.Dynamic.Core` позволяет строить LINQ-запросы из **строк** во время выполнения, например: `queryable.Where("Age >= @0 AND City == @1", 18, "London").OrderBy("Name")`. Внутри библиотека реализует собственный **парсер выражений** (лексер + разбор грамматики, похожей на C#/VB-выражения) и транслятор AST этого парсера **в стандартный `System.Linq.Expressions` `Expression Tree`** — то есть на выходе всё равно получается обычный `Expression<Func<T,bool>>`/аналог, полностью совместимый с `IQueryable`-провайдерами (EF Core и т.д.). Отличие от ручного построения через голый Expression API (группа 4/9) — не нужно писать код построения дерева самостоятельно для каждого динамического сценария; достаточно передать текстовую строку условия, что удобно для UI-конструкторов фильтров, отчётов, конфигурируемых бизнес-правил.

**Ресурсы:** [System.Linq.Dynamic.Core (GitHub)](https://github.com/zzzprojects/System.Linq.Dynamic.Core) · [System.Linq.Dynamic.Core README](https://github.com/zzzprojects/System.Linq.Dynamic.Core/blob/master/README.md)

---

### 101. [I] Приведите пример использования `System.Linq.Dynamic.Core` для построения `Where`/`OrderBy`/`Select` из строк.

**Ответ.**
```csharp
using System.Linq.Dynamic.Core;

var result = dbContext.People
    .Where("Age >= @0 && City == @1", 18, "London")
    .OrderBy("LastName, FirstName desc")
    .Select("new (FirstName, LastName, Age)")
    .ToDynamicList();
```
Обратите внимание: `Select("new (...)")` строит динамический анонимный тип (через `System.Linq.Dynamic.Core.DynamicClassFactory`, который использует `Reflection.Emit` для генерации класса на лету) — результат возвращается как `IEnumerable<dynamic>`/`DynamicClass`, а не строго типизированный `TDto`, что удобно для отчётов с произвольным набором колонок, задаваемым пользователем.

**Ресурсы:** [System.Linq.Dynamic.Core — DynamicQueryableExtensions](https://github.com/zzzprojects/System.Linq.Dynamic.Core/blob/master/src/System.Linq.Dynamic.Core/DynamicQueryableExtensions.cs)

---

### 102. [A] Какие риски безопасности несёт Dynamic LINQ, если строка запроса приходит напрямую от конечного пользователя (например, из поля поиска в UI)?

**Ответ.** Хотя парсер Dynamic LINQ строже, чем произвольный `eval`, он всё же позволяет ссылаться на члены типов (свойства, иногда методы) по имени из строки — без ограничений это потенциально даёт доступ к нежелательной функциональности (вызов «дорогих» методов, доступ к служебным/непубличным членам в зависимости от конфигурации, атаки типа ReDoS через сложные выражения). Библиотека предоставляет настройки для ограничения: whitelisting типов и методов, отключение доступа к определённым конструкциям, ограничение сложности выражения. Общая рекомендация — никогда не передавать «сырую» пользовательскую строку напрямую в `Where(...)` без валидации/ограничения допустимого набора полей и операторов (аналогично защите от SQL-инъекций на уровне текстовых SQL-запросов).

**Ресурсы:** [System.Linq.Dynamic.Core — Security considerations](https://github.com/zzzprojects/System.Linq.Dynamic.Core/blob/master/README.md)

---

### 103. [I] Чем `System.Linq.Dynamic` (без `.Core`) отличается от `System.Linq.Dynamic.Core`?

**Ответ.** `System.Linq.Dynamic` — исходная, более старая библиотека (пример от Microsoft для .NET Framework 4.0), помеченная как **deprecated**. `System.Linq.Dynamic.Core` — активно поддерживаемый форк/переписанная версия для .NET Standard/.NET Core (и современных .NET), с расширенной поддержкой синтаксиса, более полной обработкой ошибок и совместимостью с современными провайдерами (EF Core и др.). При выборе библиотеки для нового проекта следует использовать именно `.Core`-версию.

**Ресурсы:** [System.Linq.Dynamic (deprecated)](https://github.com/zzzprojects/System.Linq.Dynamic) · [System.Linq.Dynamic.Core](https://github.com/zzzprojects/System.Linq.Dynamic.Core)

---

### 104. [A] Как Dynamic LINQ обрабатывает параметры `@0`, `@1` в строке запроса и почему это важно для предотвращения инъекций?

**Ответ.** Синтаксис `@0`, `@1` ссылается на позиционные аргументы, переданные отдельно от строки-шаблона (`Where("Age >= @0", 18)`) — библиотека вставляет их в дерево как `ConstantExpression`/параметризованное значение, а не как часть текста, который парсится заново. Это прямой аналог параметризованных SQL-запросов (`WHERE Age >= @p0`) и главный механизм защиты: значения, полученные от пользователя, должны передаваться именно через параметры `@N`, а не подставляться прямой конкатенацией строк в сам шаблон запроса (`"Age >= " + userInput` — плохая практика, эквивалентная классической SQL-инъекции по духу проблемы).

**Ресурсы:** [System.Linq.Dynamic.Core README — примеры](https://github.com/zzzprojects/System.Linq.Dynamic.Core/blob/master/README.md)

---

### 105. [G] Как под капотом Dynamic LINQ строит `Select("new (FirstName, LastName)")` — динамический тип на лету — и с чем это концептуально связано (Reflection.Emit/DynamicMethod)?

**Ответ.** Для проекции в анонимоподобный тип с произвольным набором свойств, не известным на этапе компиляции, библиотека генерирует **новый CLR-тип во время выполнения** через `System.Reflection.Emit` (`TypeBuilder`, создающий класс с нужными автосвойствами) — механизм, концептуально родственный тому, что использует `LambdaExpression.Compile()` для генерации метода (вопрос 58), только здесь генерируется не метод, а целый **тип**. Далее строится `MemberInitExpression`/`NewExpression`, ссылающийся на конструктор и свойства этого динамически созданного типа, который используется как обычное дерево проекции в `Select`. Сгенерированные типы кэшируются по структуре (набору имён+типов свойств), чтобы повторные одинаковые по форме проекции не создавали новый тип каждый раз.

**Ресурсы:** [System.Linq.Dynamic.Core — DynamicClassFactory (исходники)](https://github.com/zzzprojects/System.Linq.Dynamic.Core) · [API: System.Reflection.Emit.TypeBuilder](https://learn.microsoft.com/dotnet/api/system.reflection.emit.typebuilder)

---
## Группа 11: Ограничения expression trees и эволюция C#

### 106. [I] Перечислите полный список конструкций C#, которые **нельзя** выразить как expression tree (по официальному списку MS).

**Ответ.** Согласно документации Microsoft, компилятор не построит дерево для лямбды, содержащей:
1. Условно-компилируемые методы (conditional methods, удаляемые препроцессором).
2. Доступ через `base` (`base.Method()`).
3. Выражения метод-групп, включая `&`(address-of) метод-группы и анонимные методы через `delegate`.
4. Ссылки на локальные функции.
5. Операторы-инструкции (statement bodied expressions), включая присваивание `=`.
6. Частичные методы (`partial`) без реализации (только определение).
7. Небезопасные операции с указателями (`unsafe`/pointer types).
8. Операции `dynamic`.
9. Операторы `??`/`??=` с `null`/`default` литералом слева, `null`-coalescing assignment, а также null-conditional оператор `?.`.

Плюс отдельно: statement-лямбды целиком (блок с `{ }`) и `async`/`await`-выражения.

**Ресурсы:** [MS Docs: Limitations](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/#limitations)

---

### 107. [I] Почему `async`/`await` нельзя выразить как expression tree?

**Ответ.** `async`/`await` компилируются в сложную конечную машину состояний (state machine) с продолжениями (continuations), захватом контекста синхронизации, обработкой исключений через `Task`/`ValueTask` — это категория конструкций управления потоком выполнения, принципиально отличная от «выражения, вычисляющего значение», на которой построена модель Expression Trees. Officially: если нужна асинхронная логика в динамическом сценарии, следует работать с объектами `Task` напрямую (например, строить цепочки `ContinueWith`), а не пытаться представить `async`-код как дерево.

**Ресурсы:** [MS Docs: Expression trees — data that defines code, Limitations](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-explained)

---

### 108. [A] Почему null-conditional оператор (`?.`) и null-coalescing assignment (`??=`) не поддерживаются в expression trees?

**Ответ.** Это осознанное ограничение стабильности API: `System.Linq.Expressions` — это открытый публичный контракт, которым пользуются десятки сторонних библиотек (провайдеров LINQ, mocking-фреймворков), интерпретирующих деревья вручную через `ExpressionVisitor`. Если бы каждая новая синтаксическая фича C# (начиная с C# 6, где появился `?.`) вводила **новый тип узла** дерева, это было бы breaking change для всех существующих визиторов, не знающих о новом типе узла (их `default`-ветка `switch`/непереопределённый `VisitXxx` сработала бы неправильно или бросила исключение). Поэтому команда C# выбрала стратегию: новые фичи языка либо (а) представляются в дереве через **уже существующие, более старые узлы** там, где это семантически эквивалентно, либо (б) полностью исключаются из возможности быть выраженными в дереве, если эквивалента без нового типа узла нет.

**Ресурсы:** [MS Docs: Limitations](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/#limitations)

---

### 109. [A] Общий принцип: почему «Expression trees don't support new expression node types» и как это влияет на разработку C# уже много лет?

**Ответ.** Это дословная формулировка из официальной документации: с определённого момента (фактически — после стабилизации API в .NET Framework 4/4.5) команда языка договорилась **не добавлять новые типы узлов** в `System.Linq.Expressions`, чтобы не ломать всех потребителей, которые пишут код вида `if (node is BinaryExpression) ... else if (node is MethodCallExpression) ...` без `default`-обработки неизвестных типов. Практическое следствие: многие возможности языка, появившиеся после C# 6 (records, pattern matching, switch expressions, target-typed new, required members, некоторые виды tuple-конструкций и др.), либо **не имеют** прямого представления в expression tree, либо представлены через комбинацию старых, уже существующих узлов там, где это возможно (например, некоторые кортежные конструкции представляются через существующие `NewExpression` с системным типом `ValueTuple`).

**Ресурсы:** [MS Docs: Limitations](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/#limitations)

---

### 110. [I] Может ли expression tree содержать вызов локальной функции (local function)?

**Ответ.** Нет — согласно списку ограничений, «References to local functions» явно исключены. Причина технически похожа на ограничение метод-групп: локальная функция компилируется в скрытый метод класса (или статический метод, или метод замыкания, в зависимости от захвата состояния), и компилятор C# не поддерживает автоматическое построение `MethodCallExpression`, ссылающегося на такой сгенерированный метод, из выражения-лямбды. Обходной путь — вынести логику в обычный (не локальный) метод и ссылаться на него через `MethodInfo`/вызов внутри лямбды, аналогично тому, как это делается для метод-групп (вопрос 14).

**Ресурсы:** [MS Docs: Limitations — Local functions](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/#limitations) · [MS Docs: Local functions](https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/local-functions)

---

### 111. [I] Что происходит с новым синтаксисом C# (например, target-typed `new()`) внутри expression tree — компилятор запрещает его или «переписывает» в старый эквивалент?

**Ответ.** Официальная документация формулирует общий принцип: «многие возможности, добавленные начиная с C# 6, не появляются точно в том виде, в каком написаны, в expression trees. Вместо этого новые возможности представляются в эквивалентном, более раннем синтаксисе, где это возможно». То есть если новая синтаксическая форма семантически идентична более старой конструкции, уже представимой существующими типами узлов, компилятор «незаметно» разворачивает её в эту старую форму на этапе построения дерева (так что визитор, написанный до появления новой фичи языка, продолжает корректно работать с деревом, не зная о новом синтаксисе). Если же эквивалента нет — конструкция просто недоступна в expression tree (ошибка компиляции).

**Ресурсы:** [MS Docs: Limitations](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/#limitations)

---

### 112. [A] Почему нельзя вызвать саму себя (рекурсивно) внутри `Expression<TDelegate>`, построенного из C#-лямбды напрямую по имени?

**Ответ.** Чтобы лямбда сослалась «на саму себя» по имени переменной (`Func<int,int> fact = n => n <= 1 ? 1 : n * fact(n-1);`), переменная `fact` должна существовать и быть присвоенной **до** завершения построения выражения — но при компиляции в `Expression<TDelegate>` тело строится как данные (вызовы `Expression.*`), и ссылка на `fact` внутри тела в виде обычной C#-переменной работала бы как захват **ещё не присвоенной** (или, для локальной функции — недопустимой) переменной. Технически, если `fact` объявлена заранее (например, как поле или через `Func<int,int> fact = null; fact = n => ...`), C#-компилятор всё равно не построит `Expression<TDelegate>` из такой лямбды с корректной самоссылкой автоматически для сложных случаев — рекурсию в чистом Expression API реализуют вручную через технику, аналогичную Y-комбинатору, либо просто строят рекурсию на уровне обычного делегата, а не дерева.

**Ресурсы:** [MS Docs: Interpret expressions — factorial limitations](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-interpreting#extending-this-sample)

---

### 113. [G] Как записи (records) и их синтаксис (`with`-выражения, позиционные параметры) представлены (или не представлены) в expression trees?

**Ответ.** Records в основном компилируются в обычные классы/структуры с автогенерированными конструкторами, свойствами `init`, `Equals`/`GetHashCode`/`ToString` — обращение к их свойствам и вызов их конструктора в лямбде укладываются в существующие узлы (`NewExpression`, `MemberExpression`), поэтому базовое использование records внутри expression-лямбд (`x => x.SomeRecordProp.Value`) работает без проблем. Однако **`with`-выражения** (`record with { Prop = newValue }`) — это специфическая синтаксическая конструкция, компилирующаяся в вызов специального «clone»-метода плюс присвоения через `init`-сеттеры; такая комбинация операций **не является expression-совместимой** в общем случае (она ближе к statement-семантике), и попытка использовать `with` внутри `Expression<TDelegate>`-лямбды не компилируется. Это яркий пример того, как ограничение «нет новых типов узлов» продолжает действовать и для современных фич языка.

**Ресурсы:** [MS Docs: Limitations](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/#limitations) · [MS Docs: Records](https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/record)

---

### 114. [A] Как паттерн-матчинг (`is Person { Age: > 18 }`) соотносится с expression trees?

**Ответ.** Простейшие формы паттерн-матчинга, семантически эквивалентные существующим узлам (`is SomeType` → `TypeBinaryExpression`, простое `is null`/`is not null` в некоторых контекстах), могут компилироваться в дерево, разворачиваясь в уже существующие конструкции. Но сложные property/positional-паттерны (`is Person { Age: > 18, Name: not null }`), switch-выражения с паттернами и `when`-условиями в общем случае **не имеют** прямого представления через набор существующих узлов Expression API и приводят к ошибке компиляции при попытке использовать их в expression-лямбде. Это же в явном виде упоминается в документации по LINQ-запросам как причина избегать паттерн-матчинга в `IQueryable`-запросах даже там, где формально что-то компилируется — не все провайдеры одинаково это интерпретируют (см. вопрос 75).

**Ресурсы:** [MS Docs: Write C# LINQ queries — null patterns warning](https://learn.microsoft.com/dotnet/csharp/linq/get-started/write-linq-queries#handle-null-values-in-query-expressions) · [MS Docs: Patterns](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/patterns)

---

### 115. [I] Компилятор C# генерирует дерево только из **однострочной** (expression-bodied) лямбды. Работает ли это же ограничение для expression-bodied членов класса (`public int Sum => a + b;`)?

**Ответ.** Нет, это разные механизмы. Expression-bodied member (`=>` в объявлении метода/свойства класса) — это просто альтернативный **синтаксис объявления обычного члена**, компилятор превращает его в обычный IL-метод/геттер, никакого отношения к `System.Linq.Expressions` он не имеет и никогда не строит дерево. Ограничение «только expression-лямбда» касается исключительно случая, когда лямбда-выражение (`x => ...`) присваивается переменной статического типа `Expression<TDelegate>` — именно этот конкретный контекст запускает специальный режим компиляции «строить дерево вместо IL».

**Ресурсы:** [MS Docs: Expression-bodied members](https://learn.microsoft.com/dotnet/csharp/programming-guide/statements-expressions-operators/expression-bodied-members) · [MS Docs: Limitations](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/#limitations)

---
## Группа 12: Замыкания (closures) в expression trees

### 116. [A] Как компилятор C# представляет захваченную (captured) локальную переменную внутри expression tree?

**Ответ.** Когда лямбда, компилируемая в `Expression<TDelegate>`, ссылается на локальную переменную из окружающего метода, компилятор (как и для обычных делегатов-замыканий) генерирует скрытый класс — «display class» (обычно с именем вида `<>c__DisplayClass0_0`) с публичным полем на каждую захваченную переменную. Само выражение внутри дерева получает не `ConstantExpression` со значением переменной, а `MemberExpression`, где `Expression` — это `ConstantExpression`, хранящий **ссылку на экземпляр объекта-замыкания**, а `Member` — `FieldInfo` соответствующего поля этого класса. Из-за этого при печати такого дерева (`ToString()`) видно нечто вроде `value(Program+<>c__DisplayClass0_0).threshold` вместо просто числа.

**Диаграмма.**
```
int threshold = 18;
Expression<Func<Person,bool>> e = p => p.Age > threshold;

Дерево:
  BinaryExpression (GreaterThan)
    ├─ MemberExpression (p.Age)
    └─ MemberExpression (Field: threshold)
          └─ Expression: ConstantExpression
                 Value: <экземпляр <>c__DisplayClass0_0>
                 Type: <>c__DisplayClass0_0
```

**Ресурсы:** [MS Docs: Execute expression trees — Caveats](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-execution#caveats)

---

### 117. [I] Почему захваченная переменная в expression tree — потенциальный источник ошибок трансляции в EF Core, но при этом обычно всё «просто работает»?

**Ответ.** EF Core специально умеет распознавать паттерн «`MemberExpression` над `ConstantExpression`, представляющий поле display-класса», и корректно транслирует его в параметр SQL-запроса (`@__threshold_0`), «разворачивая» значение через рефлексию во время построения SQL. Это стандартная, хорошо поддерживаемая ситуация. Проблемы возникают, когда захваченный объект — не простое значение (`int`, `string`), а сложный объект/коллекция с состоянием, которое провайдер не умеет корректно интерпретировать как параметр (например, попытка захватить и использовать в выражении экземпляр сервиса с методами, не имеющими SQL-эквивалента) — тогда трансляция падает так же, как и для любого другого неподдерживаемого `MethodCallExpression` (см. вопрос 80).

**Ресурсы:** [EF Core: How query processing works](https://learn.microsoft.com/ef/core/querying/how-query-works)

---

### 118. [A] Разберите классический MS-пример с `IDisposable`-ресурсом, захваченным в `using`-блоке, который приводит к `ObjectDisposedException` при вызове скомпилированного делегата.

**Ответ.**
```csharp
private static Func<int, int> CreateBoundResource()
{
    using (var constant = new Resource()) // Resource : IDisposable
    {
        Expression<Func<int, int>> expression = (b) => constant.Argument + b;
        var rVal = expression.Compile();
        return rVal;
    } // Dispose() вызывается здесь — до того, как rVal когда-либо будет вызван
}
```
Делегат `rVal` замкнул ссылку на объект `constant` — это происходит **на этапе построения дерева и его компиляции**, значение `constant.Argument` **не** «замораживается» в момент `Compile()` (в отличие от `ConstantExpression`, представляющей известное на этапе построения значение) — обращение к свойству `Argument` — это `MemberExpression`, вычисляемое заново **при каждом вызове** делегата. Поскольку `Dispose()` вызывается ещё до возврата из метода, а свойство `Argument` внутри `Resource` бросает `ObjectDisposedException`, если ресурс уже освобождён, — итог: вызов `rVal(5)` где-то позже в коде падает с ошибкой, которая по духу является ошибкой compile-time области видимости, но фактически проявляется в рантайме. Документация MS прямо называет это «странным, но ожидаемым» поведением мира expression trees.

**Ресурсы:** [MS Docs: Execute expression trees — Caveats (полный пример)](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-execution#caveats)

---

### 119. [I] Как избежать проблемы из предыдущего вопроса на практике?

**Ответ.** Общих универсальных правил документация не даёт («слишком много вариаций проблемы»), но практические рекомендации: (1) не захватывать `IDisposable`-объекты в выражениях, которые могут пережить область видимости `using`; (2) если экземпляр нужен, захватывать не сам ресурс, а уже извлечённое из него **значение** (скопировать `int argument = constant.Argument;` до `using`-блока и захватывать `argument`, а не `constant`); (3) быть особенно осторожным при возврате скомпилированного делегата из публичного API наружу метода, где владение временем жизни объектов, на которые он ссылается, перестаёт быть очевидным для вызывающего кода; (4) писать модульные тесты, которые вызывают делегат уже **после** предполагаемого выхода из области видимости связанных ресурсов, чтобы поймать такие баги на этапе разработки.

**Ресурсы:** [MS Docs: Execute expression trees — Caveats](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-execution#caveats)

---

### 120. [A] Может ли захват переменной `this` (доступ к состоянию текущего объекта) в expression tree, возвращаемом через публичный API, привести к утечке или неожиданным зависимостям?

**Ответ.** Да. Если метод класса создаёт и возвращает `Expression<TDelegate>`, обращающийся к `this.someField`/`this.SomeMethod()`, компилятор аналогично оборачивает `this` как захваченную переменную (обычно напрямую, без отдельного display-класса, если это единственный захват, но принцип тот же — ссылка на экземпляр объекта попадает в `ConstantExpression`). Это означает: (1) весь объект (не только нужное поле) удерживается в памяти, пока живо дерево/делегат — потенциальная утечка памяти, если предполагалось, что выражение «независимо» от исходного объекта; (2) если сборка mutable-состояния изменится после построения дерева, но до его выполнения, поведение делегата изменится соответственно (дерево видит **текущее** состояние объекта на момент вызова, а не «замороженное» на момент построения) — это может быть как желаемым, так и неожиданным поведением в зависимости от намерений автора.

**Ресурсы:** [MS Docs: Execute expression trees — Caveats](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-execution#caveats)

---

### 121. [G] Как визитор может «замкнуть» (constant-fold) захваченные переменные во время трансформации дерева — заменить `MemberExpression` над display-классом на буквальный `ConstantExpression`?

**Ответ.** Часто нужно, чтобы дерево, отправляемое во внешнюю систему (например, для сериализации в виде значений — см. группу про сериализацию), содержало не ссылки на closure-объекты, а вычисленные буквальные значения. Это делается визитором, который для `MemberExpression`, чей `Expression`-предок в конечном счёте сводится к `ConstantExpression` (т.е. вся цепочка не содержит `ParameterExpression`), **вычисляет** значение через `Expression.Lambda(node).Compile().DynamicInvoke()` и заменяет узел на `Expression.Constant(вычисленноеЗначение, node.Type)`:
```csharp
protected override Expression VisitMember(MemberExpression node)
{
    if (IsClosedOverConstant(node)) // нет ParameterExpression внутри поддерева
    {
        var value = Expression.Lambda(node).Compile().DynamicInvoke();
        return Expression.Constant(value, node.Type);
    }
    return base.VisitMember(node);
}
```
Это ровно тот приём, который используют многие ORM/провайдеры для «упрощения» дерева перед основной трансляцией — уменьшает количество узлов, которые нужно уметь распознавать транслятору, сводя все «внешние по отношению к параметрам запроса» части к простым константам.

**Ресурсы:** [API: ExpressionVisitor.VisitMember](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expressionvisitor.visitmember) · [MS Docs: Translate expression trees](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-translating)

---

### 122. [I] Почему одна и та же лямбда, вызванная в цикле с изменяющейся локальной переменной цикла (`for`/`foreach`), в старых версиях C# могла давать неожиданные результаты при захвате в выражениях?

**Ответ.** Это тот же классический баг с захватом переменных цикла, что и для обычных делегатов/замыканий: до C# 5 переменная цикла `foreach` была **одна и та же** на все итерации (переиспользовалась), поэтому все построенные внутри цикла expression trees, захватывающие эту переменную, ссылались на **одно и то же** поле display-класса — и после завершения цикла все они «видели» финальное значение переменной, а не то, что было на момент их создания. Начиная с C# 5 семантика `foreach` изменена — переменная итерации создаётся заново на каждой итерации, что устранило эту категорию ошибок для `foreach` (но не для `for` с явной внешней переменной — там поведение «одна переменная на все итерации» сохраняется, так как это соответствует явно написанному коду).

**Ресурсы:** [MS Docs: Execute expression trees — Caveats (общий класс проблем захвата)](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-execution#caveats)

---

### 123. [A] Как влияет захват переменных на кэшируемость (структурное сравнение) expression trees в контексте EF Core query plan cache?

**Ответ.** Поскольку захваченные переменные представлены как `MemberExpression` над **одним и тем же типом** display-класса (структура дерева не зависит от конкретных значений переменных, только от самого факта, что «здесь параметризованное значение такого-то типа»), EF Core может кэшировать скомпилированный SQL-план по **форме** дерева, подставляя разные значения захваченных переменных как разные значения SQL-параметров без пересборки плана — именно это делает параметризацию (вопрос 79) эффективной с точки зрения кэша: изменение значения `threshold` между вызовами не создаёт нового «структурного» ключа кэша, а вот изменение **структуры** запроса (другой набор условий, другая цепочка методов) — создаёт.

**Ресурсы:** [EF Core: How query processing works — caching](https://learn.microsoft.com/ef/core/querying/how-query-works)

---
## Группа 13: Структурное сравнение деревьев

### 124. [A] Почему `expr1 == expr2` (или `expr1.Equals(expr2)`) для двух `Expression`-объектов почти всегда возвращает `false`, даже если деревья визуально «одинаковые»?

**Ответ.** `Expression` и все его наследники не переопределяют `Equals`/`GetHashCode` для структурного (семантического) сравнения — по умолчанию используется сравнение **по ссылке** (унаследованное от `object`), то есть `expr1.Equals(expr2)` истинно, только если это буквально один и тот же объект в памяти. Два независимо построенных дерева, представляющих одну и ту же логику (`x => x.Age > 18`, построенное дважды), всегда будут разными объектами на каждом уровне (разные `ParameterExpression`, разные `ConstantExpression` и т.д.), поэтому `==`/`Equals` вернёт `false`.

**Ресурсы:** [API: Expression Class](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expression)

---

### 125. [A] Как реализовать собственное структурное сравнение двух expression trees «вручную»?

**Ответ.** Нужен рекурсивный компаратор, обходящий оба дерева параллельно и сравнивающий на каждом уровне: `NodeType`, `Type`, а также специфичные для конкретного типа узла свойства (`Method` для `MethodCallExpression`, `Member` для `MemberExpression`, `Value` для `ConstantExpression`), плюс — что особенно тонко — **связывание параметров**: если оба дерева ссылаются на «свой» `ParameterExpression` в одной и той же позиции, они должны считаться эквивалентными, даже будучи разными объектами (иначе вообще никакие два независимо построенных дерева с параметрами никогда не совпадут). Именно поэтому такой компаратор поддерживает внутреннее сопоставление «параметр A из дерева 1 соответствует параметру B из дерева 2» (аналог alpha-эквивалентности в лямбда-исчислении).

```
Псевдо-логика:
Compare(a, b, paramMap):
  if a.NodeType != b.NodeType or a.Type != b.Type: false
  switch по конкретному типу узла:
    ParameterExpression: paramMap[a] == b  (или a == b, если не связаны параметром)
    ConstantExpression:  Equals(a.Value, b.Value)
    BinaryExpression:    Compare(a.Left,b.Left) && Compare(a.Right,b.Right)
    ...
```

**Ресурсы:** [MS Docs: Execute expression trees — Caution про сравнение](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-execution#execution-and-lifetimes)

---

### 126. [A] Что такое `ExpressionEqualityComparer` в EF Core и зачем он используется внутри?

**Ответ.** EF Core (и связанные внутренние компоненты) используют собственные реализации структурного сравнения деревьев (в публичном пространстве имён есть, например, `Microsoft.EntityFrameworkCore.Query.ExpressionEqualityComparer` в некоторых версиях/сборках) для задач, требующих понять «эквивалентны ли по форме два дерева» — прежде всего для **кэша скомпилированных запросов** (вопрос 87): чтобы решить, можно ли переиспользовать ранее скомпилированный SQL для нового запроса, нужно сравнить структуру нового дерева со структурой ранее закэшированного, игнорируя конкретные значения параметризуемых констант, но учитывая типы, вызываемые методы и общую форму дерева.

**Ресурсы:** [EF Core: How query processing works — caching](https://learn.microsoft.com/ef/core/querying/how-query-works) · [API: Microsoft.EntityFrameworkCore.Query namespace](https://learn.microsoft.com/dotnet/api/microsoft.entityframeworkcore.query)

---

### 127. [I] Почему официальная документация Microsoft прямо советует **не писать** собственный универсальный «структурный diff» деревьев ради оптимизации `Compile()`?

**Ответ.** См. также вопрос 61 — прямая цитата документации: сравнение произвольных expression trees на семантическую эквивалентность — операция, время выполнения которой часто **превышает** экономию от избежания повторного `Compile()`. Кроме того, полное и корректное структурное сравнение — сложная в реализации задача (нужно учитывать альфа-эквивалентность параметров, семантически эквивалентные, но структурно разные деревья типа `a && b` vs `b && a`, и десятки типов узлов), а неполная/наивная реализация рискует давать ложноположительные совпадения (что приведёт к использованию неверно закэшированного делегата) — цена ошибки высока.

**Ресурсы:** [MS Docs: Execute expression trees — Caution](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-execution#execution-and-lifetimes)

---

### 128. [I] Как на практике кэшировать скомпилированные делегаты по «дешёвому» ключу вместо структурного сравнения дерева?

**Ответ.** Типичный практический паттерн — использовать в качестве ключа кэша не само дерево, а некий заранее известный, дешёвый в вычислении идентификатор бизнес-смысла запроса (например, enum/строку с именем фильтра, комбинацию булевых флагов «какие условия включены», хэш от списка имён полей сортировки), а не пытаться сравнивать деревья между собой:
```csharp
private static readonly ConcurrentDictionary<string, Delegate> _cache = new();

Func<T,bool> GetOrCompile<T>(string cacheKey, Func<Expression<Func<T,bool>>> build)
{
    return (Func<T,bool>)_cache.GetOrAdd(cacheKey, _ => build().Compile());
}
```
Такой подход требует, чтобы разработчик сам гарантировал, что одинаковому `cacheKey` всегда соответствует семантически одинаковое (по форме) дерево — но зато исключает дорогостоящее структурное сравнение в рантайме.

**Ресурсы:** [MS Docs: Execute expression trees — Execution and lifetimes](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-execution#execution-and-lifetimes)

---

### 129. [G] Как вычислить «структурный хэш» дерева (для использования в качестве ключа словаря), избегая при этом коллизий из-за различий в захваченных константах?

**Ответ.** Рекурсивный визитор-накопитель хэша, комбинирующий (`HashCode.Combine`) `NodeType`, `Type` и специфичные для узла метаданные (`MethodInfo`/`MemberInfo` — они сами по себе стабильно сравнимы и хэшируемы, в отличие от произвольных значений `ConstantExpression.Value`), при этом **намеренно исключающий** конкретные значения простых констант/захваченных переменных из вычисления хэша (или заменяющий их на хэш по типу, а не по значению) — именно так достигается свойство «одинаковая форма дерева → одинаковый хэш, независимо от подставленных значений», требуемое для кэша query-плана (аналог вопроса 87/126). Важно не забывать также учитывать позицию `ParameterExpression` в дереве (альфа-эквивалентность, как в вопросе 125), а не хэшировать сам объект параметра по ссылке.

**Ресурсы:** [API: HashCode.Combine](https://learn.microsoft.com/dotnet/api/system.hashcode.combine) · [EF Core: How query processing works](https://learn.microsoft.com/ef/core/querying/how-query-works)

---
## Группа 14: Производительность

### 130. [I] Каковы главные источники накладных расходов при работе с Expression Trees по сравнению с обычным «написанным руками» кодом?

**Ответ.** Три основных источника: (1) **построение дерева** — аллокации объектов-узлов (каждый вызов `Expression.Add`/`Expression.Call`/... создаёт объект в куче); для деревьев, строящихся компилятором из статических лямбд, это происходит один раз при первом достижении этой точки кода, но для деревьев, собираемых вручную в рантайме (динамические предикаты, ORM-проекции) — может происходить многократно; (2) **компиляция (`Compile()`)** — генерация IL и JIT, относительно дорогая одноразовая операция (вопрос 65); (3) **интерпретация/выполнение**, если используется режим без IL-компиляции (`preferInterpretation: true`, вопрос 66) — каждый вызов дороже, чем нативный код. Хорошо спроектированный код, использующий expression trees, минимизирует (1) и (2), выполняя их один раз и кэшируя результат, оставляя лишь дешёвый вызов скомпилированного делегата в горячем пути.

**Ресурсы:** [MS Docs: Execute expression trees](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-execution)

---

### 131. [I] Почему создание `Expression<Func<T,bool>>` из статической C#-лямбды (компилятором) не требует накладных расходов на каждый вызов метода, где она объявлена?

**Ответ.** Если лямбда не захватывает никакого состояния (или захватывает только неизменяемые/статические данные) и не содержит условной логики, зависящей от рантайма, компилятор в некоторых случаях может закэшировать построенное дерево статически (аналогично тому, как кэшируются делегаты для «чистых» статических лямбд через сгенерированное статическое поле `<>c.<>9__0_0`-подобного вида) — но в общем случае **дерево всё равно перестраивается при каждом входе в метод**, если лямбда присваивается локальной переменной внутри метода (в отличие от делегатов, для которых компилятор агрессивно кэширует статические лямбды). На практике для выражений, вычисляемых часто, рекомендуется явно выносить их построение в статические поля/свойства верхнего уровня, чтобы построение произошло один раз за время жизни приложения (домена), а не при каждом вызове метода.

**Ресурсы:** [MS Docs: Execute expression trees](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-execution)

---

### 132. [A] Как правильно кэшировать `Expression<Func<T,bool>>`/скомпилированный делегат на уровне статического поля класса-репозитория, используемого для фильтрации в EF Core?

**Ответ.**
```csharp
public static class PersonSpecs
{
    // Компилируется/строится один раз при первом обращении к типу (или явно при старте приложения)
    public static readonly Expression<Func<Person,bool>> IsAdult = p => p.Age >= 18;

    // Отдельно — если нужен именно делегат для проверки в памяти (не для передачи в IQueryable)
    private static readonly Lazy<Func<Person,bool>> _isAdultCompiled = new(() => IsAdult.Compile());
    public static Func<Person,bool> IsAdultCompiled => _isAdultCompiled.Value;
}

// Использование:
var query = db.People.Where(PersonSpecs.IsAdult);          // дерево → транслируется в SQL
bool inMemoryCheck = PersonSpecs.IsAdultCompiled(somePerson); // делегат из кэша, без повторного Compile()
```
Такой паттерн даёт «одно дерево, два способа использования» без повторных аллокаций/компиляций на каждый вызов.

**Ресурсы:** [MS Docs: Execute expression trees — Execution and lifetimes](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-execution#execution-and-lifetimes)

---

### 133. [I] Почему `MethodInfo.Invoke` в горячем цикле — плохая идея, и как Expression Trees помогают это обойти?

**Ответ.** `MethodInfo.Invoke` при каждом вызове проходит через довольно дорогой путь: проверки безопасности/доступа, упаковку value-типов аргументов в `object[]`, распаковку результата — это существенно медленнее прямого вызова. Классическая оптимизация — **один раз** построить `Expression.Call(methodInfo, ...)`, обернуть в `Expression.Lambda`, вызвать `Compile()`, и дальше использовать типизированный делегат вместо повторных `MethodInfo.Invoke` — это переносит стоимость рефлексии с «каждого вызова» на «один раз при инициализации», после чего каждый последующий вызов идёт как обычный вызов делегата (сопоставимо по скорости с прямым вызовом метода).

**Ресурсы:** [API: MethodInfo.Invoke](https://learn.microsoft.com/dotnet/api/system.reflection.methodbase.invoke) · [MS Docs: Execute expression trees](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-execution)

---

### 134. [A] Как этот приём («компилировать геттер/сеттер свойства через Expression один раз») применяется в сериализаторах и ORM для быстрого доступа к свойствам по имени?

**Ответ.** Многие высокопроизводительные библиотеки (сериализаторы, мапперы, легковесные ORM) вместо `PropertyInfo.GetValue`/`SetValue` (медленных из-за рефлексии на каждый вызов) один раз для каждого свойства строят и компилируют геттер/сеттер через Expression API:
```csharp
static Func<T,object?> BuildGetter<T>(PropertyInfo prop)
{
    var instance = Expression.Parameter(typeof(T), "instance");
    var propertyAccess = Expression.Property(instance, prop);
    var convert = Expression.Convert(propertyAccess, typeof(object));
    return Expression.Lambda<Func<T,object?>>(convert, instance).Compile();
}
```
Полученный делегат кэшируется (обычно в `Dictionary<PropertyInfo, Delegate>` при инициализации типа) и переиспользуется для всех последующих обращений — по скорости это близко к прямому обращению к свойству в коде и на порядки быстрее рефлексии на каждый вызов.

**Ресурсы:** [API: Expression.Property](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expression.property) · [API: PropertyInfo.GetValue](https://learn.microsoft.com/dotnet/api/system.reflection.propertyinfo.getvalue)

---

### 135. [I] Почему компилированный из Expression Tree делегат обычно так же быстр, как и «нативно написанный» код, но не всегда абсолютно идентичен по производительности?

**Ответ.** После `Compile()` + JIT итоговый машинный код исполняется так же, как любой другой .NET-метод — принципиальной разницы в модели исполнения нет. Возможные небольшие расхождения: (1) компилятор C#, компилируя обычный код, иногда применяет дополнительные оптимизации на уровне генерации IL (например, определённые паттерны инлайнинга констант), которые `LambdaCompiler` внутри `Expression.Compile()` не всегда воспроизводит один в один; (2) для делегатов, полученных из `Compile()`, иногда чуть отличается способ передачи `this`/захваченных переменных (через явный объект-замыкание, а не через обычные локальные переменные метода), что может незначительно повлиять на паттерны доступа к памяти. На практике эти различия обычно пренебрежимо малы по сравнению с самой логикой бизнес-кода, но при экстремальной оптимизации горячих путей стоит профилировать конкретный случай.

**Ресурсы:** [MS Docs: Execute expression trees](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-execution)

---

### 136. [A] Как компиляция запросов EF Core (`EF.CompileQuery`) связана с темой производительности expression trees?

**Ответ.** `EF.CompileQuery`/`EF.CompileAsyncQuery` — явный механизм, позволяющий разработчику **заранее**, один раз, скомпилировать LINQ-запрос (включая всю трансляцию в SQL, а не только `Expression.Compile()`) в переиспользуемый делегат, минуя даже стандартный автоматический query cache EF Core (вопрос 87) на каждый вызов — экономится время на поиск в кэше и на сопоставление структуры дерева. Это полезно для очень «горячих», часто вызываемых запросов, где даже накладные расходы стандартного пайплайна (включая хэширование дерева для поиска в кэше) заметны на профиле производительности приложения.

**Ресурсы:** [EF Core: Advanced performance topics — Compiled queries](https://learn.microsoft.com/ef/core/performance/advanced-performance-topics) · [API: EF.CompileQuery](https://learn.microsoft.com/dotnet/api/microsoft.entityframeworkcore.ef.compilequery)

---

### 137. [I] Почему динамическая сборка LINQ-запроса «с нуля» на каждый HTTP-запрос (без кэширования по форме) может деградировать производительность приложения при масштабировании?

**Ответ.** Если структура дерева (не значения параметров, а именно форма — набор вызванных методов Where/Select/Join) меняется на каждый вызов (например, из-за динамического добавления условий фильтрации в цикле без нормализации порядка/группировки условий), кэш скомпилированных query-планов EF Core (вопрос 87) не сможет переиспользовать предыдущие результаты — каждый уникальный по форме запрос заново проходит через весь дорогой конвейер трансляции в SQL. При высокой нагрузке это проявляется как рост CPU-потребления и задержек именно в компонентах компиляции запроса, а не в самой БД — типичная причина: неупорядоченное добавление условий (`if`) в разном порядке для разных запросов, из-за чего структурно эквивалентные по смыслу, но разные по порядку построения деревья считаются разными для кэша.

**Ресурсы:** [EF Core: How query processing works — caching](https://learn.microsoft.com/ef/core/querying/how-query-works)

---

### 138. [A] Как соотносится производительность выражений с интерпретацией (`preferInterpretation: true`) против стандартной IL-компиляции для очень коротко живущих (once-off) деревьев?

**Ответ.** Если дерево строится и используется **ровно один раз** (например, для однократного вычисления динамически собранного условия в скрипте/конфигурации, которое больше никогда не вызовется), стоимость полной IL-компиляции + JIT может значительно превышать стоимость самого выполнения — в этом случае режим интерпретации (обход дерева напрямую, без генерации кода) часто оказывается быстрее «в сумме» (build + execute), хотя каждый отдельный «шаг» выполнения интерпретируемого дерева медленнее эквивалентного скомпилированного кода. Правило пальца: чем больше раз будет вызван один и тот же скомпилированный делегат, тем выгоднее полная IL-компиляция; для единичных вызовов интерпретация может быть предпочтительнее, но конкретный порог стоит подтверждать бенчмарком (BenchmarkDotNet), а не общими рассуждениями.

**Ресурсы:** [API: Expression<TDelegate>.Compile(bool preferInterpretation)](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expression-1.compile)

---

### 139. [I] Какую роль в производительности играет размер (глубина/число узлов) expression tree при трансляции ORM-провайдером?

**Ответ.** Время работы конвейера трансляции (визиторы, проходящие по дереву несколько раз на разных этапах — навигационное разворачивание, SQL-трансляция, генерация текста) в целом растёт с числом узлов дерева — очень большие, глубоко вложенные LINQ-запросы (десятки условий, множественные `Join`/`Include`, вложенные подзапросы) заметно увеличивают время самой трансляции (не выполнения SQL на сервере, а именно построения SQL-текста на клиенте), что особенно заметно при первом (некэшированном) выполнении запроса такой формы. Практический совет для очень сложных динамически строящихся запросов — измерять именно этот этап (например, через `.ToQueryString()` с замером времени, либо через встроенные диагностические счётчики EF Core), а не полагаться на интуицию о том, «где узкое место».

**Ресурсы:** [EF Core: How query processing works](https://learn.microsoft.com/ef/core/querying/how-query-works) · [EF Core: Diagnostics](https://learn.microsoft.com/ef/core/logging-events-diagnostics/)

---
## Группа 15: Практические сценарии: AutoMapper, Moq, Specification

### 140. [I] Как FluentValidation использует `RuleFor(x => x.Email)` — при чём здесь expression trees?

**Ответ.** `RuleFor<T,TProperty>(Expression<Func<T,TProperty>> expression)` принимает дерево, а не делегат, чтобы: (1) извлечь **имя свойства** (через `MemberExpression.Member.Name`) для формирования сообщения об ошибке по умолчанию («'Email' must not be empty») и для ключа ошибки, используемого, например, при биндинге к конкретному полю формы на клиенте; (2) построить компилированный геттер (аналогично вопросу 134) для быстрого многократного получения значения свойства при валидации множества объектов. Если бы API принимал делегат (`Func<T,TProperty>`), библиотека потеряла бы возможность узнать «как называется проверяемое свойство», не прибегая к дополнительным строковым параметрам.

**Ресурсы:** [FluentValidation (NuGet)](https://www.nuget.org/packages/FluentValidation) · [API: MemberExpression](https://learn.microsoft.com/dotnet/api/system.linq.expressions.memberexpression)

---

### 141. [I] Как ASP.NET Core Razor `@Html.DisplayNameFor(m => m.Email)`/строго типизированные HTML-хелперы используют выражения?

**Ответ.** Аналогичный паттерн: `Expression<Func<TModel,TValue>>` передаётся не для выполнения, а чтобы статически (по дереву) извлечь имя свойства модели (`m.Email` → `MemberExpression.Member.Name` = `"Email"`) и, опционально, атрибуты над этим свойством (`[Display(Name = "...")]` через рефлексию над `MemberInfo`). Такой подход даёт типобезопасность и поддержку рефакторинга (переименование `Email` в IDE автоматически обновит и вызов хелпера), в отличие от передачи имени свойства строкой напрямую (`Html.DisplayNameFor("Email")`), которую IDE не отследит при переименовании.

**Ресурсы:** [MS Docs: Tag Helpers vs HTML Helpers](https://learn.microsoft.com/aspnet/core/mvc/views/overview) · [API: Expression<TDelegate>](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expression-1)

---

### 142. [A] Разберите, как Moq различает `.Setup(x => x.GetValue())` (без аргументов) от `.Setup(x => x.GetValue(It.IsAny<int>()))` с точки зрения дерева.

**Ответ.** Оба случая дают `MethodCallExpression`, но с разным содержимым `Arguments`. Для первого — пустой список аргументов (метод без параметров). Для второго — один элемент в `Arguments`, сам являющийся `MethodCallExpression`, ссылающимся на статический метод `It.IsAny<int>()`. Moq анализирует каждый элемент `Arguments`: если это распознаваемый вызов из семейства `It.*` (`IsAny`, `Is`, `IsInRange`, `IsRegex` и т.д.), он конвертируется во внутренний matcher-объект; если это произвольное другое выражение (в т.ч. `ConstantExpression` с конкретным значением) — оно трактуется как требование точного совпадения аргумента по значению (через `Equals`). Именно поэтому `It.IsAny<int>()` **нельзя** просто «вызвать» отдельно от контекста `Setup` — сам метод при обычном вызове возвращает `default(int)`, вся «магия» происходит на уровне анализа дерева.

**Ресурсы:** [Moq (NuGet)](https://www.nuget.org/packages/Moq) · [API: MethodCallExpression.Arguments](https://learn.microsoft.com/dotnet/api/system.linq.expressions.methodcallexpression.arguments)

---

### 143. [A] Опишите архитектуру Specification pattern с комбинаторами `And`/`Or`/`Not`, полностью основанную на expression trees, для использования и в SQL, и в памяти.

**Ответ.**
```csharp
public interface ISpecification<T>
{
    Expression<Func<T,bool>> ToExpression();
}

public static class SpecificationExtensions
{
    public static ISpecification<T> And<T>(this ISpecification<T> left, ISpecification<T> right) =>
        new AndSpecification<T>(left, right);
}

public class AndSpecification<T> : ISpecification<T>
{
    private readonly ISpecification<T> _left, _right;
    public AndSpecification(ISpecification<T> left, ISpecification<T> right) { _left = left; _right = right; }

    public Expression<Func<T,bool>> ToExpression()
    {
        var leftExpr = _left.ToExpression();
        var rightExpr = _right.ToExpression();
        var param = leftExpr.Parameters[0];
        var rightBody = new ReplaceParameterVisitor(rightExpr.Parameters[0], param).Visit(rightExpr.Body);
        return Expression.Lambda<Func<T,bool>>(Expression.AndAlso(leftExpr.Body, rightBody), param);
    }
}
```
Такая архитектура — прямое комбинирование уже рассмотренных техник: подмены параметра (вопрос 50/92) для объединения независимо построенных деревьев в композитную структуру, декларативно отражающую бизнес-правила и одинаково пригодную и для `IQueryable.Where`, и для `spec.ToExpression().Compile()` над объектом в памяти.

**Ресурсы:** [LINQKit](https://github.com/scottksmith95/LINQKit) · [MS Docs: Build dynamic queries](https://learn.microsoft.com/dotnet/csharp/linq/how-to-build-dynamic-queries)

---

### 144. [I] Как Dapper/легковесные ORM могут использовать expression trees, не будучи полноценным `IQueryable`-провайдером?

**Ответ.** В отличие от EF Core, Dapper сам по себе не строит SQL из LINQ-выражений — но экосистема вокруг него (например, `Dapper.Contrib`) и многие библиотеки быстрого маппинга результатов SQL-запроса на объекты (materializers) используют expression trees для **быстрого построения объектов из `DataReader`/строк результата**: вместо `Activator.CreateInstance` + `PropertyInfo.SetValue` в цикле по колонкам (медленно из-за рефлексии), библиотека один раз для каждого типа результата строит и компилирует делегат-«материализатор» через `Expression.New`/`Expression.Bind`/`Expression.MemberInit`, читающий значения из `IDataRecord` напрямую по индексам колонок — это тот же паттерн «скомпилировать один раз, вызывать много раз», что и для геттеров/сеттеров (вопрос 134), применённый к целому конструктору объекта.

**Ресурсы:** [API: Expression.MemberInit](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expression.memberinit) · [MS Docs: Build expression trees](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-building)

---

### 145. [A] Как MediatR-подобные библиотеки/CQRS-фреймворки иногда используют expression trees для построения фабрик обработчиков без рефлексии в горячем пути?

**Ответ.** При разрешении DI-контейнером конкретного обработчика (`IRequestHandler<TRequest,TResponse>`) для произвольного `TRequest` во время выполнения, некоторые высокопроизводительные реализации избегают повторяющегося `Activator.CreateInstance`/рефлексионного вызова конструктора, один раз строя expression tree, представляющую вызов конструктора класса-обработчика (`Expression.New(constructorInfo, paramExpressions)`), компилируя её в фабричный делегат `Func<IServiceProvider, THandler>`, и кэшируя этот делегат по типу обработчика — при последующих запросах создание экземпляра идёт через уже скомпилированный делегат, а не заново через рефлексию. Это тот же общий паттерн ускорения создания объектов, что применяется и в DI-контейнерах (например, во внутренних оптимизациях `Microsoft.Extensions.DependencyInjection` для построения графа зависимостей).

**Ресурсы:** [API: Expression.New](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expression.new) · [MS Docs: Execute expression trees](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-execution)

---

### 146. [I] Как AutoMapper строит «плоские» проекции для вложенных объектов (`Order.Customer.Name` → `Dto.CustomerName`) на уровне дерева?

**Ответ.** Конфигурация карты (по соглашению об именовании или явно через `.ForMember(...)`) транслируется в цепочку вложенных `MemberExpression` — `MemberExpression(MemberExpression(x, "Customer"), "Name")`, что соответствует `x.Customer.Name` — этот узел затем становится значением соответствующего `MemberAssignment` внутри итогового `MemberInitExpression` результирующего `Dto`. При использовании `ProjectTo` (проекция на уровне `IQueryable`, вопрос 89) такая цепочка обращений транслируется EF Core в SQL `JOIN` с выборкой нужной колонки (`Customer.Name`), не требуя загрузки всего связанного объекта `Customer` целиком.

**Ресурсы:** [AutoMapper (NuGet)](https://www.nuget.org/packages/AutoMapper) · [API: MemberExpression](https://learn.microsoft.com/dotnet/api/system.linq.expressions.memberexpression)

---

### 147. [A] Как реализовать собственный простой «object mapper» (аналог упрощённого AutoMapper) на expression trees для двух известных на этапе компиляции типов?

**Ответ.**
```csharp
static Func<TSource,TDest> BuildMapper<TSource,TDest>() where TDest : new()
{
    var source = Expression.Parameter(typeof(TSource), "src");
    var bindings = typeof(TDest).GetProperties()
        .Where(destProp => destProp.CanWrite)
        .Select(destProp =>
        {
            var sourceProp = typeof(TSource).GetProperty(destProp.Name);
            if (sourceProp is null) return null;
            var access = Expression.Property(source, sourceProp);
            var converted = sourceProp.PropertyType == destProp.PropertyType
                ? (Expression)access
                : Expression.Convert(access, destProp.PropertyType);
            return Expression.Bind(destProp, converted);
        })
        .Where(b => b is not null)!
        .Cast<MemberBinding>();

    var newExpr = Expression.New(typeof(TDest));
    var init = Expression.MemberInit(newExpr, bindings);
    return Expression.Lambda<Func<TSource,TDest>>(init, source).Compile();
}
```
Это упрощённая, но рабочая иллюстрация того, как AutoMapper (в базовом сценарии по соглашению об именовании свойств) строит и кэширует делегаты преобразования между типами — именно поэтому AutoMapper значительно быстрее «ручного» маппинга через рефлексию на каждый вызов, но всё ещё медленнее «руками написанного» присваивания свойство-в-свойство (из-за накладных расходов на первичное построение/компиляцию, амортизируемых при многократном использовании).

**Ресурсы:** [AutoMapper (NuGet)](https://www.nuget.org/packages/AutoMapper) · [API: Expression.MemberInit](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expression.memberinit)

---

### 148. [I] Как строго типизированный доступ к именам свойств через выражения (`Expression<Func<T,object>>`) помогает при рефакторинге по сравнению со строками (`nameof` vs `"PropertyName"`)?

**Ответ.** И `nameof(T.Property)`, и `Expression<Func<T,object>>`-подход дают проверку компилятором и корректное переименование средствами IDE, но у них разное назначение: `nameof` — это просто получение **строки** с именем на этапе компиляции (без построения дерева, без рантайм-стоимости), удобно, когда достаточно самого имени. Expression-based API (как в FluentValidation/HTML-хелперах, вопросы 140-141) нужен, когда, помимо имени, требуется ещё и типобезопасный **доступ к значению/типу свойства** через одно и то же выражение (например, чтобы построить и делегат-геттер, и извлечь метаданные атрибутов над этим свойством) — то есть решает более широкую задачу, чем просто получение строки.

**Ресурсы:** [MS Docs: nameof expression](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/nameof) · [API: Expression<TDelegate>](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expression-1)

---

### 149. [G] Как объединить Specification pattern с пагинацией и сортировкой, полностью выражая query-конфигурацию через один переиспользуемый объект на expression trees?

**Ответ.** Расширенная спецификация может инкапсулировать не только `Expression<Func<T,bool>>` (критерий), но и `Expression<Func<T,object>>` (ключ сортировки) плюс параметры пагинации, применяемые единым методом-расширением к `IQueryable<T>`:
```csharp
public class Specification<T>
{
    public Expression<Func<T,bool>>? Criteria { get; init; }
    public Expression<Func<T,object>>? OrderBy { get; init; }
    public bool OrderByDescending { get; init; }
    public int? Skip { get; init; }
    public int? Take { get; init; }
}

static IQueryable<T> Apply<T>(this IQueryable<T> query, Specification<T> spec)
{
    if (spec.Criteria is not null) query = query.Where(spec.Criteria);
    if (spec.OrderBy is not null)
        query = spec.OrderByDescending ? query.OrderByDescending(spec.OrderBy) : query.OrderBy(spec.OrderBy);
    if (spec.Skip is int s) query = query.Skip(s);
    if (spec.Take is int t) query = query.Take(t);
    return query;
}
```
Такой подход — стандартная практика в DDD-ориентированных архитектурах (репозитории, принимающие `Specification<T>` вместо десятка отдельных параметров фильтрации), полностью основанная на композиции expression trees без единой строки SQL в коде приложения.

**Ресурсы:** [MS Docs: Build dynamic queries](https://learn.microsoft.com/dotnet/csharp/linq/how-to-build-dynamic-queries) · [EF Core: Querying data](https://learn.microsoft.com/ef/core/querying/)

---
## Группа 16: Отладка expression trees

### 150. [I] Что напечатает `expression.ToString()` для простого дерева, и насколько это надёжно для отладки сложных деревьев?

**Ответ.** Каждый узел `Expression` переопределяет `ToString()`, выдавая приближённое к C#-синтаксису представление — например, `x => (x.Age > 18)`. Это удобно для быстрой визуальной проверки простых выражений, но у формата есть ограничения: (1) он **не является** валидным исходным C#-кодом, который можно скопировать и скомпилировать обратно (особенно для сложных узлов вроде `MemberInitExpression`, `BlockExpression`, замыканий — см. вопрос 116, где вывод выглядит как `value(Namespace.Class+<>c__DisplayClass0_0).field`); (2) он не показывает типы узлов (`NodeType`) явно — для полноценной отладки структуры дерева часто нужны более специализированные инструменты (следующие вопросы).

**Ресурсы:** [API: Expression.ToString](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expression.tostring)

---

### 151. [A] Как использовать отладчик Visual Studio (DebuggerDisplay/Watch) для пошагового просмотра структуры дерева во время выполнения?

**Ответ.** В окне Watch/QuickWatch можно раскрывать узел `Expression` — видны его свойства (`NodeType`, `Type`, а для конкретного подтипа, если привести через `as`/явное приведение в Watch-выражении, — специфичные свойства вроде `Left`/`Right`/`Method`/`Arguments`). Полезный приём — в Watch-окне писать выражение с явным приведением типа: `((BinaryExpression)node).Left`, чтобы обойти то, что статический тип переменной — базовый `Expression`, а нужные детали доступны только у конкретного наследника. Для действительно сложных деревьев такой ручной пошаговый обход быстро становится утомительным — отсюда популярность визуализаторов (следующие вопросы).

**Ресурсы:** [MS Docs: Using the debugger — Watch window](https://learn.microsoft.com/visualstudio/debugger/watch-and-quickwatch-windows)

---

### 152. [A] Что такое сторонние библиотеки/визуализаторы вроде `ExpressionTreeToString` и какую проблему они решают?

**Ответ.** Такие библиотеки (доступны как NuGet-пакеты) рекурсивно обходят произвольное дерево и генерируют читаемое, часто многострочное текстовое представление, показывающее **и** приближённый C#-код, **и** явную древовидную структуру с типами узлов (`NodeType`) на каждом уровне — то, чего не даёт стандартный `ToString()`. Некоторые из них также умеют экспортировать структуру в формат для визуализации графов (например, DOT для Graphviz), что особенно полезно при отладке очень больших деревьев, порождаемых сложными LINQ-запросами EF Core, где текстового представления в одну строку уже недостаточно для понимания вложенности.

**Ресурсы:** [NuGet: поиск "ExpressionTreeToString"](https://www.nuget.org/packages?q=ExpressionTreeToString) · [MS Docs: Expression Trees](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/)

---

### 153. [I] Как в Visual Studio использовать «Debugger Visualizer» специально для Expression Trees?

**Ответ.** Начиная с определённых версий Visual Studio, для объектов типа `Expression` доступен встроенный визуализатор (значка «лупа»/«диаграмма» рядом со значением в отладчике), показывающий дерево в виде интерактивной иерархической схемы вместо плоского текста — это заметно ускоряет понимание структуры сложных, глубоко вложенных выражений (например, целого LINQ-запроса с несколькими `Join`/`Where`/`Select`) по сравнению с чтением `ToString()`-вывода. Если встроенного визуализатора нет в конкретной версии IDE, можно установить сторонние расширения из Visual Studio Marketplace, реализующие аналогичную функциональность.

**Ресурсы:** [MS Docs: Visual Studio debugger visualizers](https://learn.microsoft.com/visualstudio/debugger/create-custom-visualizers-of-data)

---

### 154. [I] Как получить и прочитать финальный SQL, сгенерированный EF Core из LINQ-запроса, без выполнения запроса к реальной БД?

**Ответ.** Метод `IQueryable<T>.ToQueryString()` (доступен начиная с EF Core 5) возвращает строку с итоговым SQL-текстом (с плейсхолдерами параметров), **не выполняя** сам запрос — удобно для отладки трансляции прямо в юнит-тестах или интерактивно в отладчике. Дополнительно можно включить полное логирование через `optionsBuilder.LogTo(Console.WriteLine, LogLevel.Information)` и `EnableSensitiveDataLogging()` (только для разработки — раскрывает значения параметров в логах, что небезопасно для production) для просмотра фактически выполненных SQL-команд с реальными значениями параметров.

**Ресурсы:** [EF Core: ToQueryString](https://learn.microsoft.com/ef/core/querying/how-query-works) · [EF Core: Logging](https://learn.microsoft.com/ef/core/logging-events-diagnostics/simple-logging)

---

### 155. [G] Как написать собственный минимальный «pretty-printer» для дерева, показывающий отступы по уровням вложенности (для интеграции в собственные диагностические логи)?

**Ответ.**
```csharp
class PrettyPrintVisitor : ExpressionVisitor
{
    private readonly StringBuilder _sb = new();
    private int _depth;

    public string Print(Expression e) { Visit(e); return _sb.ToString(); }

    public override Expression? Visit(Expression? node)
    {
        if (node is null) return null;
        _sb.Append(' ', _depth * 2).AppendLine($"{node.NodeType} : {node.Type.Name}");
        _depth++;
        var result = base.Visit(node);
        _depth--;
        return result;
    }
}
```
Переопределение именно центрального диспетчера `Visit(Expression)` (вопрос 57), а не отдельных `VisitXxx`, гарантирует, что отступ и печать сработают **для всех** типов узлов единообразно, включая любые новые/редкие типы, не требуя явной поддержки каждого из них.

**Ресурсы:** [API: ExpressionVisitor.Visit](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expressionvisitor.visit)

---
## Группа 17: Сериализация expression trees

### 156. [A] Почему `System.Linq.Expressions` не предоставляет встроенной сериализации (например, через `System.Text.Json`) для произвольных деревьев?

**Ответ.** Основная причина — деревья могут содержать объекты, принципиально не сериализуемые в общем виде: `MethodInfo`/`PropertyInfo`/`ConstructorInfo` (ссылки на метаданные сборок, зависящие от точной версии/расположения сборки на целевой машине), делегаты и замыкания над произвольными объектами (включая, например, соединения с БД, файловые хендлы), а также потенциально циклические или очень специфичные для конкретной реализации CLR структуры. Полная, универсальная и **безопасная** сериализация произвольного `Expression`-дерева — не просто вопрос формата данных, а вопрос воссоздания «указателей» на код и метаданные, которые не всегда переносимы между процессами/машинами/версиями .NET, поэтому платформа сознательно не берёт на себя эту ответственность как встроенную возможность.

**Ресурсы:** [MS Docs: Expression Trees](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/) · [API: MethodInfo](https://learn.microsoft.com/dotnet/api/system.reflection.methodinfo)

---

### 157. [A] Какие существуют практические подходы к «квази-сериализации» expression trees, раз встроенного механизма нет?

**Ответ.** Несколько подходов, каждый со своими компромиссами:
1. **Сериализация в текстовый DSL** — как делает `System.Linq.Dynamic.Core` в обратную сторону (не сериализация готового дерева, а изначальное построение из строки, вопрос 100); для сериализации существующего дерева можно написать визитор, генерирующий такую же строку, которую потом можно распарсить обратно этой же библиотекой.
2. **Сторонние библиотеки сериализации выражений** (существуют NuGet-пакеты, специализирующиеся именно на этой задаче — сериализуют дерево в JSON/XML с описанием узлов и восстанавливают через рефлексию по строковым именам методов/типов, требуя, чтобы целевая сборка с этими типами/методами была доступна при десериализации).
3. **Сериализация не самого дерева, а его «намерения»** — например, хранение не C#-выражения, а простого DTO с именами полей/операторами/значениями (собственный мини-DSL уровня приложения), из которого дерево **строится заново** каждый раз через Expression API (это, по сути, обходит проблему сериализации целиком, перенося сериализуемость на уровень пользовательских данных, а не CLR-метаданных).

**Ресурсы:** [System.Linq.Dynamic.Core](https://github.com/zzzprojects/System.Linq.Dynamic.Core) · [MS Docs: Build dynamic queries](https://learn.microsoft.com/dotnet/csharp/linq/how-to-build-dynamic-queries)

---

### 158. [I] Почему подход №3 (сериализация «намерения», а не дерева) обычно считается наиболее надёжным для реальных production-систем?

**Ответ.** Потому что он избегает всех перечисленных в вопросе 156 проблем: DTO с полями `{ Field: "Age", Operator: "GreaterThan", Value: 18 }` — это обычные, полностью сериализуемые данные (без ссылок на `MethodInfo`/сборки/замыкания), которые можно безопасно передавать между сервисами, хранить в БД/конфигурации, версионировать. Восстановление в expression tree происходит уже на принимающей стороне через контролируемый, ограниченный (whitelisted) построитель, который сам решает, как транслировать `"Age" + "GreaterThan" + 18` в `Expression.GreaterThan(Expression.Property(param, "Age"), Expression.Constant(18))` — с полным контролем над безопасностью (какие поля/операторы разрешены) и совместимостью версий (в отличие от сериализации самого дерева, зависящей от точных версий сборок на обеих сторонах).

**Ресурсы:** [MS Docs: Build dynamic queries](https://learn.microsoft.com/dotnet/csharp/linq/how-to-build-dynamic-queries)

---

### 159. [A] Какие проблемы возникнут при попытке напрямую сериализовать `Expression` через `System.Text.Json`/`BinaryFormatter` без специальной обработки?

**Ответ.** `System.Text.Json` по умолчанию сериализует публичные свойства объекта — попытка сериализовать, например, `MethodCallExpression`, приведёт к попытке сериализовать его свойство `Method` (`MethodInfo`), которое, в свою очередь, содержит ссылки на `Module`/`Assembly`/`DeclaringType` и глубокий граф объектов рефлексии, не предназначенный для JSON-сериализации (циклические ссылки, отсутствие публичного конструктора для восстановления, платформенно-зависимые внутренние поля) — на практике это либо выбросит исключение, либо создаст бесполезный, невосстанавливаемый JSON. `BinaryFormatter` в современных версиях .NET и вовсе объявлен небезопасным и удалён/заблокирован по умолчанию начиная с .NET 5+/9 (CVE-класс уязвимостей десериализации) — использовать его для чего-либо, включая expression trees, не рекомендуется.

**Ресурсы:** [MS Docs: BinaryFormatter security](https://learn.microsoft.com/dotnet/standard/serialization/binaryformatter-security-guide) · [API: System.Text.Json](https://learn.microsoft.com/dotnet/api/system.text.json)

---

### 160. [G] Как передать «предикат фильтрации» между микросервисами (например, через очередь сообщений), не сериализуя expression tree напрямую?

**Ответ.** Практическая архитектура: сервис-инициатор описывает фильтр через собственный сериализуемый контракт (например, простое AST в формате JSON: `{"and": [{"field":"age","op":"gte","value":18}, {"field":"city","op":"eq","value":"London"}]}`), пересылает его как обычные данные, а принимающий сервис имеет **собственный, локальный** транслятор этого JSON-AST в `Expression<Func<T,bool>>` через Expression API (комбинация подходов из вопросов 90 и 158) — с собственным whitelisting допустимых полей `T` для этого конкретного эндпоинта. Такой подход также даёт естественную защиту от версионной рассинхронизации: если на принимающей стороне изменилась модель `T`, транслятор просто не найдёт поле и вернёт понятную ошибку валидации, а не крэш при десериализации несовместимого бинарного/CLR-специфичного формата.

**Ресурсы:** [MS Docs: Build dynamic queries](https://learn.microsoft.com/dotnet/csharp/linq/how-to-build-dynamic-queries) · [System.Linq.Dynamic.Core](https://github.com/zzzprojects/System.Linq.Dynamic.Core)

---

### 161. [I] Можно ли сериализовать (сохранить) уже **скомпилированный** делегат (`Func<T,bool>`) для последующего использования, минуя повторную сборку дерева?

**Ответ.** Нет — делегат ссылается на исполняемый метод в памяти конкретного, уже загруженного модуля/сборки текущего процесса (для `Compile()` — на динамически сгенерированный, находящийся только в памяти метод, никогда не сохранявшийся на диск, начиная с .NET Core, где убрана возможность `AssemblyBuilder.Save`, см. вопрос 59) — такую ссылку в принципе невозможно сериализовать в переносимом виде и восстановить в другом процессе/на другой машине. Единственный практический путь «сохранить и переиспользовать логику между процессами» — сохранять исходные **данные**, из которых дерево/делегат можно **заново построить и скомпилировать** на принимающей стороне (см. вопросы 157-160), а не пытаться сериализовать сам скомпилированный код.

**Ресурсы:** [MS Docs: Execute expression trees — Lambda expressions to functions](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-execution#lambda-expressions-to-functions)

---
## Группа 18: Expression Trees vs Reflection.Emit vs Source Generators

### 162. [A] Чем построение кода через `Expression` API принципиально отличается от прямой работы с `System.Reflection.Emit` (`ILGenerator`)?

**Ответ.** `Reflection.Emit` — низкоуровневый API, требующий вручную выписывать **отдельные IL-инструкции** (`Ldarg`, `Callvirt`, `Add`, `Ret` и т.д.) через `ILGenerator.Emit(OpCodes.Xxx, ...)` — мощно, но многословно, подвержено ошибкам (несбалансированный стек операндов приводит к `InvalidProgramException` при загрузке метода, а не сразу при написании кода) и требует глубокого понимания модели выполнения CLR. `Expression` API — более высокоуровневая абстракция: вы описываете дерево на уровне «выражений C#-подобной семантики» (сложение, вызов метода, условие), а генерацию корректных, сбалансированных IL-инструкций из этого дерева берёт на себя внутренний `LambdaCompiler` при вызове `Compile()`. Итог: Expression API — как правило, быстрее в разработке и безопаснее (меньше шансов получить невалидный IL), но даёт меньше контроля над мельчайшими деталями генерируемого кода по сравнению с ручным `Reflection.Emit`.

**Ресурсы:** [API: System.Reflection.Emit.ILGenerator](https://learn.microsoft.com/dotnet/api/system.reflection.emit.ilgenerator) · [MS Docs: Execute expression trees](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-execution)

---

### 163. [A] В каких сценариях стоит выбрать прямой `Reflection.Emit` вместо Expression Trees, несмотря на бóльшую сложность?

**Ответ.** Когда нужен максимальный контроль над сгенерированным кодом и/или конструкции, которые Expression API просто не поддерживает как единый узел (например, некоторые низкоуровневые IL-паттерны, генерация целых типов с несколькими методами и сложной иерархией через `TypeBuilder`, специфичная работа с указателями/`unsafe`-кодом, которую Expression API не выражает, вопрос 106). Также прямой `Reflection.Emit` может давать чуть более компактный/оптимальный IL для очень специфичных горячих путей, где даже небольшие накладные расходы `LambdaCompiler` (который генерирует несколько более «общий», не всегда предельно оптимальный IL) имеют значение при экстремальных требованиях к производительности. На практике абсолютное большинство прикладных задач динамической генерации кода прекрасно решается через Expression API без необходимости опускаться до сырого IL.

**Ресурсы:** [API: System.Reflection.Emit.TypeBuilder](https://learn.microsoft.com/dotnet/api/system.reflection.emit.typebuilder)

---

### 164. [G] Чем `DynamicMethod` отличается от полноценной сборки через `AssemblyBuilder`/`TypeBuilder`, и какой из них использует `Expression.Compile()`?

**Ответ.** `DynamicMethod` — облегчённый механизм для создания **одного** метода «в воздухе», не привязанного к постоянному типу/сборке, оптимизированный для быстрой генерации и сборки мусора (может быть собран GC, когда больше не используется, в отличие от типов, загруженных в `AssemblyBuilder`, которые обычно живут, пока жив соответствующий `AssemblyLoadContext`). Именно `DynamicMethod` использует `Expression<TDelegate>.Compile()` внутри — это оптимально для сценария «скомпилировать одну лямбду в один делегат». `AssemblyBuilder`/`TypeBuilder`/`ModuleBuilder` — более тяжеловесная инфраструктура для создания **целых типов** с несколькими членами (полями, несколькими методами, реализацией интерфейсов) — именно её использует, например, `CompileToMethod` (когда он доступен, вопрос 59) для записи в заранее подготовленный `MethodBuilder`, и её же использует `System.Linq.Dynamic.Core` для генерации типов «на лету» под динамические проекции (вопрос 105).

**Ресурсы:** [API: System.Reflection.Emit.DynamicMethod](https://learn.microsoft.com/dotnet/api/system.reflection.emit.dynamicmethod) · [API: System.Reflection.Emit.AssemblyBuilder](https://learn.microsoft.com/dotnet/api/system.reflection.emit.assemblybuilder)

---

### 165. [A] Как современные Source Generators (C# 9+) соотносятся с Expression Trees как альтернативный подход к «динамическому» поведению?

**Ответ.** Source Generators решают концептуально похожую задачу («сгенерировать код на основе описания/метаданных без ручного написания шаблонного кода вручную»), но принципиально другим способом и на другом этапе жизненного цикла: Source Generator работает **во время компиляции** (анализирует синтаксис/атрибуты через Roslyn API и генерирует дополнительный **исходный C#-код**, который затем компилируется обычным компилятором как часть проекта), тогда как Expression Trees работают **во время выполнения** (дерево строится и, опционально, компилируется в делегат уже в запущенном приложении, на основе данных, которые могли быть неизвестны на этапе компиляции — например, введённых пользователем). Практическое следствие: код, сгенерированный Source Generator, статически виден в IDE, отлаживается как обычный код, не требует JIT-компиляции «на лету» и хорошо совместим с AOT/trimming; код, представленный Expression Trees, гибче (реагирует на данные времени выполнения), но требует рефлексии/`Reflection.Emit`-подобных механизмов, которые могут быть ограничены в некоторых AOT-средах.

**Ресурсы:** [MS Docs: Source Generators](https://learn.microsoft.com/dotnet/csharp/roslyn-sdk/source-generators-overview) · [MS Docs: Expression Trees](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/)

---

### 166. [G] Почему многие современные высокопроизводительные библиотеки (например, System.Text.Json с Source Generator-режимом) мигрируют от Reflection/Expression Trees к Source Generators?

**Ответ.** Три главные причины: (1) **производительность запуска (startup)** — построение и компиляция expression trees для сериализации каждого типа при первом обращении добавляет заметное время к холодному старту приложения (особенно критично для serverless/контейнеризированных нагрузок с частыми холодными стартами); Source Generator-код уже полностью скомпилирован в IL заранее, без затрат на построение/JIT дерева в рантайме. (2) **AOT-совместимость** — Native AOT (`dotnet publish -p:PublishAot=true`) не поддерживает динамическую генерацию кода через `Reflection.Emit`/`Expression.Compile()` в полном объёме (JIT недоступен в AOT-скомпилированном приложении) — Source Generator-код, будучи обычным статически скомпилированным C#, полностью совместим с AOT. (3) **Trimming/размер приложения** — статический анализ кода, сгенерированного Source Generator, точнее для trimmer'а (удаления неиспользуемого кода), чем динамически используемая через рефлексию логика, которую trimmer не всегда может безопасно проанализировать. При этом Expression Trees остаются незаменимыми там, где логика **действительно** неизвестна на этапе компиляции (динамические предикаты, ORM-запросы на основе рантайм-модели) — Source Generator не может сгенерировать код для условия, которое ещё не существует на момент сборки.

**Ресурсы:** [MS Docs: Native AOT deployment](https://learn.microsoft.com/dotnet/core/deploying/native-aot) · [MS Docs: System.Text.Json source generation](https://learn.microsoft.com/dotnet/standard/serialization/system-text-json/source-generation)

---

### 167. [A] Работает ли `Expression<TDelegate>.Compile()` в средах Native AOT/trimmed-приложений?

**Ответ.** Это существенное ограничение: полноценная динамическая компиляция через `Compile()` (генерация IL «на лету» с последующим JIT) требует наличия JIT-компилятора в рантайме — в чистом Native AOT-приложении (где весь код скомпилирован заранее в нативный машинный код, а JIT отсутствует) `Expression.Compile()` для деревьев, требующих генерации нового кода, либо не работает вовсе, либо (в зависимости от версии рантайма и конкретной реализации fallback-интерпретатора) выполняется в режиме **интерпретации** (вопрос 66) вместо полноценной IL-компиляции — с соответствующими накладными расходами на каждый вызов. Библиотеки, активно полагающиеся на Expression Trees (некоторые ORM, DI-контейнеры, сериализаторы в «классическом» режиме) требуют особого внимания при переносе на Native AOT — часто у них есть отдельный, AOT-совместимый режим работы (например, через Source Generators, вопрос 166).

**Ресурсы:** [MS Docs: Native AOT deployment — Limitations](https://learn.microsoft.com/dotnet/core/deploying/native-aot) · [API: Expression<TDelegate>.Compile](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expression-1.compile)

---

### 168. [G] Приведите пример, когда правильным архитектурным решением будет **гибрид**: Source Generator для известной на этапе компиляции части + Expression Trees для действительно динамической части.

**Ответ.** Типичный пример — библиотека валидации/маппинга, где 95% правил известны заранее (статические свойства класса, известные атрибуты) — для них Source Generator на этапе компиляции генерирует прямой, быстрый, AOT-совместимый код без единой рефлексии. Но 5% сценариев требуют динамического поведения — например, правила, конфигурируемые администратором через UI и хранящиеся в БД (аналог вопроса 160), или маппинг с типами, известными только в рантайме через плагинную систему — для этой части всё ещё используется Expression Trees (построение дерева из конфигурации + `Compile()`), с осознанием, что этот путь не будет работать в чистом AOT-режиме без fallback на интерпретацию. Такая гибридная архитектура даёт баланс: максимальная производительность и AOT-совместимость там, где возможно, и гибкость там, где она действительно нужна.

**Ресурсы:** [MS Docs: Source Generators](https://learn.microsoft.com/dotnet/csharp/roslyn-sdk/source-generators-overview) · [MS Docs: Expression Trees](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/)

---

### 169. [I] Какие практические плюсы у Expression Trees сохраняются даже с учётом развития Source Generators?

**Ответ.** Source Generators работают только с информацией, доступной **на этапе компиляции** (синтаксис, атрибуты, статические типы проекта) — они принципиально не могут решить задачи, где сама логика становится известна только во время выполнения: пользовательские динамические фильтры (группа 9), конфигурация бизнес-правил, читаемая из БД/файла после старта приложения, построение запросов на основе метаданных, полученных через reflection над сборками, загруженными во время выполнения (плагинная архитектура). Expression Trees остаются единственным (в рамках стандартной модели .NET, без выхода за пределы управляемого кода) способом безопасно и гибко превратить «данные, описывающие код» в реально исполняемый код именно во время выполнения приложения.

**Ресурсы:** [MS Docs: Expression Trees](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/) · [MS Docs: Build dynamic queries](https://learn.microsoft.com/dotnet/csharp/linq/how-to-build-dynamic-queries)

---
## Группа 19: Nullable, boxing и `Convert`

### 170. [I] Как представлено сравнение `Nullable<T>` (`int?`) с обычным значением в expression tree?

**Ответ.** `p.NullableAge > 18` строится как `BinaryExpression(GreaterThan)`, где `Left` — `MemberExpression` типа `int?` (`Nullable<int>`), а `Right` — `ConstantExpression` типа `int` (или уже приведённая к `int?` через неявный `Convert`, в зависимости от того, как компилятор разрешил операцию). `BinaryExpression` для операций над nullable-типами имеет свойство `IsLiftedToNull`/`IsLifted` — признак того, что это «поднятая» (lifted) операция: если один из операндов равен `null`, вся операция сравнения обычно возвращает `false` (для операторов сравнения) согласно правилам C# для nullable-типов, и `LambdaCompiler` при `Compile()` корректно воспроизводит эту семантику через дополнительные проверки на `HasValue` в сгенерированном IL.

**Ресурсы:** [API: BinaryExpression.IsLifted](https://learn.microsoft.com/dotnet/api/system.linq.expressions.binaryexpression.islifted) · [MS Docs: Nullable value types](https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/nullable-value-types)

---

### 171. [A] Что означают свойства `IsLifted` и `IsLiftedToNull` у `BinaryExpression`?

**Ответ.** `IsLifted` — `true`, если оператор был «поднят» для работы с nullable-операндами (то есть базовая операция определена для non-nullable типов, но применена к `Nullable<T>`). `IsLiftedToNull` — `true`, если результирующий тип самой операции — тоже nullable (например, `int? + int?` даёт `int?`, а не `int`, — это `IsLiftedToNull == true`; тогда как `int? == int?` даёт обычный `bool`, не `bool?` — для таких операций `IsLifted == true`, но `IsLiftedToNull == false`). Эти флаги важны при написании собственных интерпретаторов/трансляторов дерева (например, кастомного LINQ-провайдера, группа 21) — без их учёта легко неправильно воспроизвести null-семантику C# при генерации целевого кода/SQL.

**Ресурсы:** [API: BinaryExpression](https://learn.microsoft.com/dotnet/api/system.linq.expressions.binaryexpression)

---

### 172. [I] Как в дереве представлен боксинг (упаковка value-типа в `object`) — например, при передаче `int` в `Expression<Func<T,object>>`?

**Ответ.** Боксинг представлен как `UnaryExpression` с `NodeType == ExpressionType.Convert`, где `Operand.Type` — value-тип (например, `int`), а `Type` самого узла — `object` (или другой ссылочный тип-«приёмник»). Пример: `Expression<Func<Person,object>> e = p => p.Age;` (где `Age` — `int`) — тело будет `UnaryExpression(Convert, MemberExpression(p.Age))` с `Type == typeof(object)`. При компиляции `LambdaCompiler` генерирует соответствующую IL-инструкцию `box` для этого узла — то есть боксинг «виден» в дереве явно, как отдельный узел, а не является неявной, скрытой операцией, как в обычном C#-коде (где компилятор вставляет `box` автоматически без отдельного синтаксиса).

**Ресурсы:** [API: UnaryExpression](https://learn.microsoft.com/dotnet/api/system.linq.expressions.unaryexpression) · [API: Expression.Convert](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expression.convert)

---

### 173. [A] В чём разница между `Expression.Convert` и `Expression.ConvertChecked`?

**Ответ.** `Expression.Convert(operand, type)` создаёт узел приведения типа **без** проверки переполнения — аналог обычного `(int)someDouble` в неявном `unchecked`-контексте (стандартном для C# по умолчанию): если значение не помещается в целевой тип, результат «молча» переполняется/усекается по стандартным правилам CLR. `Expression.ConvertChecked(operand, type)` создаёт узел, при компиляции генерирующий IL-инструкцию с проверкой переполнения (аналог `checked((int)someDouble)`) — при переполнении на этапе выполнения будет выброшено `OverflowException`. Выбор между ними важен при построении деревьев, эмулирующих код, написанный в `checked`-контексте (например, внутри блока `checked { }` в C#, который компилируется именно в `ConvertChecked`/`AddChecked` и т.п.).

**Ресурсы:** [API: Expression.ConvertChecked](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expression.convertchecked) · [MS Docs: checked and unchecked](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/checked-and-unchecked)

---

### 174. [I] Как в дереве представлено приведение вниз по иерархии (down-cast, `(Manager)person`) и как это отличается от `TypeAs`?

**Ответ.** `(Manager)person` (явное приведение через круглые скобки) компилируется в `UnaryExpression(Convert, ...)` с `Type == typeof(Manager)` — при неудачном приведении во время выполнения выбрасывается `InvalidCastException` (как и в обычном C#-коде). `person as Manager` (оператор `as`) компилируется в отдельный узел `UnaryExpression(TypeAs, ...)` — при неудачном приведении возвращается `null`, исключение не выбрасывается. Хотя оба представлены через `UnaryExpression`, разный `NodeType` (`Convert` против `TypeAs`) сообщает транслятору/визитору совершенно разную ожидаемую семантику обработки ошибок, которую нужно учитывать при написании собственного интерпретатора дерева.

**Ресурсы:** [API: Expression.TypeAs](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expression.typeas) · [API: UnaryExpression](https://learn.microsoft.com/dotnet/api/system.linq.expressions.unaryexpression)

---

### 175. [A] Как EF Core транслирует `p.NullableAge.HasValue`/`p.NullableAge.Value` в SQL, ведь `HasValue`/`Value` — обычные вызовы методов/свойств `Nullable<T>`?

**Ответ.** EF Core (как и большинство реляционных провайдеров) распознаёт `MemberExpression` с `Member.Name == "HasValue"` над выражением типа `Nullable<T>` как специальный случай и транслирует его в SQL-проверку `IS NOT NULL`, а `.Value`/`.GetValueOrDefault()` — как обращение непосредственно к значению колонки (SQL сам по себе не различает «nullable int» и «int» на уровне выражений — только на уровне ограничений колонки), не пытаясь буквально вызвать метод `Nullable<T>.GetValueOrDefault` как обычный C#-метод. Это ещё один пример того, что провайдер имеет **собственный список распознаваемых паттернов** поверх стандартных узлов Expression API — по сути, «словарь» соответствий между конкретными комбинациями `MethodCallExpression`/`MemberExpression` и целевыми SQL-конструкциями.

**Ресурсы:** [EF Core: How query processing works](https://learn.microsoft.com/ef/core/querying/how-query-works) · [MS Docs: Nullable value types](https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/nullable-value-types)

---
## Группа 20: Ошибки построения и компиляции

### 176. [I] Какое исключение бросается, если попытаться создать `BinaryExpression` (например, `Expression.Add`) для типов, для которых операция не определена, и как его прочитать?

**Ответ.** Выбрасывается `System.InvalidOperationException` с сообщением вида *«The binary operator Add is not defined for the types 'System.String' and 'System.DateTime'.»* — то есть Expression API на этапе **построения** узла (а не только при компиляции/выполнении) проверяет совместимость типов операндов для стандартных операторов, и явно называет и операцию, и оба типа в тексте сообщения, что обычно сразу указывает на ошибку (например, забытое явное приведение типа одного из операндов, см. вопрос 41).

**Ресурсы:** [API: Expression.Add Method](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expression.add) · [API: BinaryExpression](https://learn.microsoft.com/dotnet/api/system.linq.expressions.binaryexpression)

---

### 177. [I] Что за исключение «variable ... of type ... referenced from scope '', but it is not defined», и как его исправить?

**Ответ.** Это `InvalidOperationException`, возникающая при попытке скомпилировать/собрать `Expression.Lambda`, тело которого ссылается на `ParameterExpression`, **не включённый** в список `Parameters` этой лямбды (или включённый как другой объект-ссылка с тем же именем/типом, см. вопрос 19). Классическая причина — создание двух разных `ParameterExpression` с одинаковым именем `"x"` в разных частях кода построения дерева и случайное использование не того экземпляра при финальной сборке `Expression.Lambda(body, wrongParam)`. Исправление — убедиться, что везде, где параметр используется внутри `body`, использован **тот же самый** объект `ParameterExpression`, который передан в список параметров `Lambda`.

**Ресурсы:** [API: Expression.Lambda Method](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expression.lambda) · [API: ParameterExpression](https://learn.microsoft.com/dotnet/api/system.linq.expressions.parameterexpression)

---

### 178. [A] Что произойдёт, если `Expression.Lambda<TDelegate>` вызвать с `body`, чей `Type` не совпадает с `TDelegate`'s return type?

**Ответ.** Будет выброшено `ArgumentException` на этапе построения (например: *«ExpressionType must be Boolean to test true/false or Invalid operation»* или *«Incorrect number of type args»*/сообщение о несовпадении типа возврата в зависимости от конкретной причины) — компилятор Expression API строго проверяет соответствие `Body.Type` типу, который бы вернул `TDelegate.Invoke` при выполнении. Например, попытка построить `Expression.Lambda<Func<int,bool>>(constIntExpr, param)`, где `constIntExpr.Type == typeof(int)`, а не `bool`, немедленно бросит исключение — нужно либо изменить сам body на выражение правильного типа, либо явно обернуть его в `Expression.Convert(body, typeof(bool))`, если приведение вообще имеет смысл.

**Ресурсы:** [API: Expression.Lambda<TDelegate> Method](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expression.lambda#system-linq-expressions-expression-lambda-1%28system-linq-expressions-expression-system-collections-generic-ienumerable%28%28system-linq-expressions-parameterexpression%29%29%29)

---

### 179. [I] Какая ошибка возникает при `Expression.Call`, если сигнатура переданных аргументов не совпадает с параметрами `MethodInfo`, и как её отладить?

**Ответ.** `ArgumentException` (обычно с сообщением вида *«Expression of type 'System.Int32' cannot be used for parameter of type 'System.String' of method '...'»*) — Expression API проверяет соответствие количества и типов аргументов реальной сигнатуре метода. Частые причины: (1) метод перегружен, и `GetMethod` вернул не ту перегрузку (нужно уточнить сигнатуру вторым аргументом `GetMethod(name, new[] { typeof(...) , ... })`, см. вопрос 36); (2) забыто явное приведение типа одного из аргументов (`Expression.Convert`) при несовпадении, например, `int` vs `int?`; (3) для методов расширения — забыт сам объект-приёмник как первый аргумент (см. вопрос 24). Отладка — вывести `method.ToString()`/`method.GetParameters()` и сравнить с фактически переданными `Expression`-аргументами по типам.

**Ресурсы:** [API: Expression.Call Method](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expression.call) · [API: MethodInfo.GetParameters](https://learn.microsoft.com/dotnet/api/system.reflection.methodbase.getparameters)

---

### 180. [A] Что произойдёт при попытке `Expression.Property(instance, "NonExistentProperty")`, и как избежать этого класса ошибок при динамическом построении по строковым именам?

**Ответ.** Будет выброшено `ArgumentException` («Instance property 'NonExistentProperty' is not defined for type '...'») — во время выполнения, а не компиляции, что особенно опасно при построении предикатов из пользовательского ввода (группа 9/10). Практическая защита — явно проверять существование свойства до вызова `Expression.Property` (например, через `type.GetProperty(name)` и проверку на `null` с осмысленным сообщением об ошибке валидации, а не давать «просочиться» «сырому» `ArgumentException` до конечного пользователя), а ещё лучше — валидировать имя свойства по заранее определённому белому списку допустимых для фильтрации полей (см. также вопрос 97).

**Ресурсы:** [API: Expression.Property Method](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expression.property)

---

### 181. [I] Какое исключение бросает `Compile()`, если дерево ссылается на приватный член, недоступный в контексте вызывающего кода?

**Ответ.** В зависимости от конкретной ситуации это может быть `MemberAccessException`/`FieldAccessException`/`MethodAccessException` («Attempt by method '...' to access field '...' failed») — Expression API само по себе позволяет **построить** узел, ссылающийся на приватный член (через рефлексию с `BindingFlags.NonPublic`), но при **компиляции** в реальный исполняемый код через `Compile()` применяются обычные правила доступа CLR (JIT/verifier проверяет, что генерируемый IL-код имеет право обращаться к данному члену) — если сгенерированный динамический метод формально не имеет достаточных прав доступа к приватному члену другого типа, будет выброшено исключение доступа именно на этапе `Compile()`/первого вызова, а не на этапе построения дерева.

**Ресурсы:** [API: LambdaExpression.Compile](https://learn.microsoft.com/dotnet/api/system.linq.expressions.lambdaexpression.compile) · [MS Docs: Accessibility levels](https://learn.microsoft.com/dotnet/csharp/programming-guide/classes-and-structs/access-modifiers)

---
## Группа 21: Guru — пишем мини LINQ-провайдер

### 182. [G] Из каких минимальных компонентов состоит собственный `IQueryable`-провайдер?

**Ответ.** Минимально необходимы четыре компонента:
1. **`IQueryable<T>`-реализация** (например, класс `MyQueryable<T>`), хранящая `Expression` и ссылку на `IQueryProvider`.
2. **`IQueryProvider`-реализация**, реализующая `CreateQuery<TElement>`/`CreateQuery` (оборачивает новое дерево в новый `MyQueryable<TElement>`) и `Execute<TResult>`/`Execute` (запускает реальную трансляцию + выполнение и возвращает результат).
3. **Транслятор дерева** (обычно `ExpressionVisitor`-наследник), обходящий накопленное дерево запроса (цепочку `MethodCallExpression` от `Where`/`Select`/...) и генерирующий целевое представление (SQL-текст, HTTP-запрос, что угодно).
4. **Исполнитель** — код, реально отправляющий сгенерированный запрос во внешнюю систему и материализующий результат обратно в объекты `T`.

**Диаграмма.**
```
MyQueryable<T>.Where(predicate)
       │  строит MethodCallExpression, оборачивающий предыдущее Expression
       ▼
 Provider.CreateQuery<T>(newExpr)  →  новый MyQueryable<T> с обновлённым Expression
       │
   ... цепочка продолжается (Select/OrderBy/...) ...
       │
  foreach / .ToList()
       ▼
 Provider.Execute<IEnumerable<T>>(finalExpr)
       │  ExpressionVisitor-транслятор → целевой запрос (например, SQL)
       ▼
  Выполнение во внешней системе → материализация → IEnumerable<T>
```

**Ресурсы:** [Matt Warren: Building an IQueryable Provider — Part I](https://learn.microsoft.com/en-us/archive/blogs/mattwar/linq-building-an-iqueryable-provider-part-i) · [API: IQueryProvider](https://learn.microsoft.com/dotnet/api/system.linq.iqueryprovider)

---

### 183. [G] Напишите скелет минимального `IQueryProvider`, транслирующего только `Where` с простыми условиями в псевдо-SQL строку.

**Ответ.**
```csharp
public class SimpleQueryProvider : IQueryProvider
{
    public IQueryable CreateQuery(Expression expression) =>
        (IQueryable)Activator.CreateInstance(
            typeof(SimpleQueryable<>).MakeGenericType(expression.Type.GetGenericArguments()[0]),
            this, expression)!;

    public IQueryable<TElement> CreateQuery<TElement>(Expression expression) =>
        new SimpleQueryable<TElement>(this, expression);

    public object? Execute(Expression expression) => Execute<object>(expression);

    public TResult Execute<TResult>(Expression expression)
    {
        var translator = new SqlTranslatingVisitor();
        translator.Visit(expression);
        string sql = translator.ToSql();
        // здесь — реальный вызов к БД/имитация выполнения sql и материализация результата
        return default!;
    }
}

public class SimpleQueryable<T> : IQueryable<T>
{
    public SimpleQueryable(IQueryProvider provider, Expression expression)
        { Provider = provider; Expression = expression; }
    public Expression Expression { get; }
    public IQueryProvider Provider { get; }
    public Type ElementType => typeof(T);
    public IEnumerator<T> GetEnumerator() =>
        ((IEnumerable<T>)Provider.Execute<IEnumerable<T>>(Expression)).GetEnumerator();
    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}
```
Это учебный минимум, иллюстрирующий связь между `IQueryable`/`IQueryProvider` — production-реализация (как в EF Core) добавляет к этому кэширование, обработку сотен видов операторов, материализацию, отслеживание изменений сущностей и многое другое.

**Ресурсы:** [Matt Warren: Building an IQueryable Provider series](https://learn.microsoft.com/en-us/archive/blogs/mattwar/linq-building-an-iqueryable-provider-series) · [IQToolkit (полная референс-реализация)](https://github.com/mattwar/iqtoolkit)

---

### 184. [G] Как транслирующий визитор должен обрабатывать `MethodCallExpression` для `Where`, извлекая имя колонки и значение условия?

**Ответ.**
```csharp
public class SqlTranslatingVisitor : ExpressionVisitor
{
    private readonly StringBuilder _sb = new();
    public string ToSql() => _sb.ToString();

    protected override Expression VisitMethodCall(MethodCallExpression node)
    {
        if (node.Method.Name == "Where" && node.Method.DeclaringType == typeof(Queryable))
        {
            Visit(node.Arguments[0]); // рекурсивно обрабатываем "источник" (может быть ещё Where/Select)
            _sb.Append(_sb.Length == 0 ? "WHERE " : " AND ");
            var lambda = (LambdaExpression)StripQuotes(node.Arguments[1]);
            Visit(lambda.Body);
            return node;
        }
        return base.VisitMethodCall(node);
    }

    protected override Expression VisitBinary(BinaryExpression node)
    {
        Visit(node.Left);
        _sb.Append(node.NodeType switch
        {
            ExpressionType.Equal => " = ",
            ExpressionType.GreaterThan => " > ",
            ExpressionType.LessThan => " < ",
            _ => throw new NotSupportedException(node.NodeType.ToString())
        });
        Visit(node.Right);
        return node;
    }

    protected override Expression VisitMember(MemberExpression node)
    {
        _sb.Append(node.Member.Name);
        return node;
    }

    protected override Expression VisitConstant(ConstantExpression node)
    {
        _sb.Append(node.Value is string s ? $"'{s}'" : node.Value);
        return node;
    }

    private static Expression StripQuotes(Expression e)
    {
        while (e.NodeType == ExpressionType.Quote) e = ((UnaryExpression)e).Operand;
        return e;
    }
}
```
Обратите внимание на `StripQuotes` — как обсуждалось в вопросе 23, лямбда внутри `Where` оборачивается в `Quote`, и её нужно явно «развернуть», прежде чем обращаться к её `Body`.

**Ресурсы:** [MS Docs: Translate expression trees](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-translating) · [Matt Warren: Building an IQueryable Provider — Part III (перевод в SQL)](https://learn.microsoft.com/en-us/archive/blogs/mattwar/linq-building-an-iqueryable-provider-part-iii)

---

### 185. [G] Почему в production-провайдере (например, EF Core) нельзя просто «конкатенировать строки» при трансляции, как в упрощённом примере выше, и что используется вместо этого?

**Ответ.** Прямая конкатенация строк SQL в процессе обхода дерева работает только для очень простых, линейных случаев — но реальные запросы требуют: (1) **параметризации** значений (не встраивать значения буквально в текст ради защиты от инъекций/переиспользования планов, вопрос 79); (2) **промежуточного, семантического представления** запроса (собственное дерево SQL-конструкций — `SelectExpression`, `WhereClause`, `JoinExpression` и т.п. — а не просто текст), потому что порядок построения частей SQL не всегда совпадает с порядком обхода C#-дерева (например, `GroupBy`/`Having`/`OrderBy` в SQL идут в определённом синтаксическом порядке независимо от порядка вызовов LINQ-операторов в коде); (3) **многоэтапной оптимизации** сгенерированного запроса до сериализации в текст (устранение лишних подзапросов, объединение условий). Именно поэтому реальные провайдеры (в т.ч. `RelationalSqlTranslatingExpressionVisitor` в EF Core, вопрос 78) сначала строят собственное промежуточное дерево представления SQL, и лишь на последнем этапе — генератор конкретного диалекта — превращают это промежуточное дерево в текст.

**Ресурсы:** [EF Core: How query processing works](https://learn.microsoft.com/ef/core/querying/how-query-works) · [RelationalSqlTranslatingExpressionVisitor](https://learn.microsoft.com/dotnet/api/microsoft.entityframeworkcore.query.relationalsqltranslatingexpressionvisitor)

---

### 186. [G] Как обрабатывать `Select` с проекцией в новый анонимный/именованный тип внутри самописного провайдера?

**Ответ.** Аналогично `Where` (вопрос 184), но вместо накопления условия `WHERE`, визитор для `Select` должен обойти `Body` лямбды, ожидая **либо** простой `MemberExpression` (проекция на одно поле — `SELECT columnName`), **либо** `NewExpression`/`MemberInitExpression` (проекция на несколько полей — нужно перечислить каждый `Argument`/`Binding` как отдельную колонку с алиасом, соответствующим имени параметра конструктора/свойства назначения). Дополнительно нужно **запомнить** саму форму проекции (список имён и типов результирующих «колонок»), чтобы на этапе материализации знать, как собрать объект результата из полученных сырых данных (строки результата SQL-запроса, JSON-ответа и т.п.) обратно в CLR-объекты — обычно для этого при трансляции параллельно строится **делегат-материализатор** (тот же паттерн, что и в вопросе 144).

**Ресурсы:** [Matt Warren: Building an IQueryable Provider — Part IV (проекции)](https://learn.microsoft.com/en-us/archive/blogs/mattwar/linq-building-an-iqueryable-provider-part-iv) · [API: MemberInitExpression](https://learn.microsoft.com/dotnet/api/system.linq.expressions.memberinitexpression)

---

### 187. [G] Как учебный провайдер должен реагировать на неподдерживаемый узел (например, вызов неизвестного метода внутри условия), и как это соотносится с поведением EF Core?

**Ответ.** Правильная практика — **явно** бросать информативное исключение (`NotSupportedException`, указывающее конкретный узел/метод, который не удалось транслировать), а не «молча» игнорировать неподдерживаемую часть или падать с невнятной `NullReferenceException` глубже по стеку. Это ровно то поведение, которое сознательно реализует EF Core начиная с версии 3.0 (вопрос 77) — предсказуемая, явная ошибка трансляции лучше, чем скрытое неверное выполнение (например, тихий пропуск условия фильтрации, что привело бы к возврату неверных, потенциально более широких данных, чем ожидал разработчик).

**Ресурсы:** [EF Core: Client vs. Server Evaluation](https://learn.microsoft.com/ef/core/querying/client-eval) · [MS Docs: Interpret expressions](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-interpreting)

---

### 188. [G] Где можно найти более полную, реалистичную (но всё ещё учебную) референс-реализацию LINQ-провайдера с генерацией SQL, join'ами и материализацией?

**Ответ.** Проект **IQToolkit** Мэтта Уоррена (доступен на GitHub) — прямое продолжение серии его блог-постов «Building an IQueryable Provider»: полноценная, хоть и учебная, реализация LINQ-провайдера к реляционным БД, включающая трансляцию сложных запросов (join, group by, агрегаты), генерацию SQL для нескольких диалектов и материализацию результатов — по сути, «мини-EF» в образовательных целях, показывающий на реальном, читаемом коде все концепции, обсуждённые в этой группе вопросов, но в куда большем масштабе, чем упрощённый пример из вопросов 183-184.

**Ресурсы:** [IQToolkit на GitHub](https://github.com/mattwar/iqtoolkit) · [Полный список блог-постов Matt Warren](https://learn.microsoft.com/en-us/archive/blogs/mattwar/linq-building-an-iqueryable-provider-series)

---

### 189. [G] Как оптимизировать самописный провайдер, добавив кэш скомпилированных «шаблонов» запроса по структуре дерева (аналог query plan cache EF Core, вопрос 87)?

**Ответ.** Общая стратегия: (1) перед трансляцией — пропустить дерево через визитор, «вымывающий» конкретные значения `ConstantExpression`/захваченные переменные в параметры-плейсхолдеры (аналог constant-folding из вопроса 121, но в обратную сторону — не сворачивание в константу, а **выделение** переменной части как параметра); (2) вычислить структурный ключ («форму») получившегося обезличенного дерева (например, через рекурсивный хэш, аналог вопроса 129); (3) если по этому ключу в кэше уже есть скомпилированный шаблон целевого запроса (SQL-текст с плейсхолдерами) — переиспользовать его, подставив актуальные значения параметров вместо повторной трансляции; (4) если нет — оттранслировать заново и сохранить в кэш по этому ключу. Именно так, в упрощённом виде, устроен реальный query plan cache в EF Core и аналогичных провайдерах.

**Ресурсы:** [EF Core: How query processing works — caching](https://learn.microsoft.com/ef/core/querying/how-query-works)

---
## Группа 22: Потокобезопасность и иммутабельность

### 190. [I] Потокобезопасны ли уже построенные expression trees для одновременного чтения из нескольких потоков?

**Ответ.** Да — благодаря неизменяемости (вопрос 5), готовое, уже построенное дерево можно безопасно **читать** (обходить, передавать визиторам, компилировать) одновременно из множества потоков без блокировок: поскольку ни один поток не может изменить состояние узла после его создания, классические проблемы гонок данных (data races) при конкурентном чтении неизменяемых объектов не возникают. Это одна из практических выгод иммутабельного дизайна API — например, статически закэшированное `static readonly Expression<Func<T,bool>>` (вопрос 132) можно безопасно шарить между потоками веб-приложения без какой-либо синхронизации.

**Ресурсы:** [MS Docs: Immutability of Expression Trees](https://learn.microsoft.com/dotnet/visual-basic/programming-guide/concepts/expression-trees/#immutability-of-expression-trees)

---

### 191. [I] Потокобезопасен ли сам вызов `Compile()` — можно ли вызывать его одновременно из разных потоков над одним и тем же деревом?

**Ответ.** Сам вызов `Compile()` не модифицирует исходное дерево (снова — иммутабельность), поэтому вызывать `expr.Compile()` из нескольких потоков **одновременно над одним и тем же объектом дерева** формально безопасно в том смысле, что не приведёт к повреждению состояния дерева. Однако это неэффективно: каждый параллельный вызов независимо выполнит полную, дорогостоящую компиляцию (вопрос 65), создав несколько разных, но эквивалентных по поведению делегатов — рекомендуется использовать `Lazy<T>` или `ConcurrentDictionary.GetOrAdd` для гарантии, что фактическая компиляция произойдёт **ровно один раз**, даже при конкурентном первом обращении из нескольких потоков.

**Ресурсы:** [API: LambdaExpression.Compile](https://learn.microsoft.com/dotnet/api/system.linq.expressions.lambdaexpression.compile) · [API: System.Lazy<T>](https://learn.microsoft.com/dotnet/api/system.lazy-1)

---

### 192. [A] Является ли скомпилированный делегат (`Func<T,bool>` из `Compile()`) потокобезопасным для параллельных вызовов?

**Ответ.** Это зависит не от факта происхождения делегата из expression tree, а исключительно от **того, что именно** он делает: если тело выражения не обращается к разделяемому изменяемому состоянию (не читает/пишет статические поля, не вызывает методы с побочными эффектами над общими объектами), сгенерированный код будет так же потокобезопасен, как и любая чистая функция, независимо от того, была ли она написана вручную или сгенерирована из дерева. Если же дерево (например, через захваченную переменную/замыкание, вопрос 116) ссылается на изменяемое общее состояние — потокобезопасность делегата определяется потокобезопасностью этого состояния, точно так же, как и для любого обычного C#-делегата с аналогичным захватом.

**Ресурсы:** [MS Docs: Execute expression trees](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-execution)

---

### 193. [A] Безопасно ли переиспользовать один и тот же объект `ParameterExpression` в нескольких **разных** независимо компилируемых деревьях одновременно (в разных потоках)?

**Ответ.** Да — поскольку `ParameterExpression`, как и любой другой узел, неизменяем, использование одного и того же объекта-параметра как части нескольких разных деревьев (что и происходит, например, при комбинировании предикатов через `PredicateBuilder`, вопрос 92, или при построении множества деревьев, разделяющих общие подвыражения ради экономии памяти) не создаёт никаких проблем потокобезопасности сам по себе — узел не «знает» и не хранит информацию о том, в скольких деревьях он используется, никакого разделяемого мутируемого состояния между использованиями нет.

**Ресурсы:** [API: ParameterExpression](https://learn.microsoft.com/dotnet/api/system.linq.expressions.parameterexpression)

---

### 194. [G] Как `ExpressionVisitor` гарантирует, что параллельный обход одного и того же дерева двумя разными визиторами (в двух потоках) не приведёт к состоянию гонки?

**Ответ.** Поскольку каждый визитор при трансформации **создаёт новые** объекты узлов там, где что-то меняется (никогда не мутирует существующие узлы «на месте», см. вопрос 56), два параллельных обхода одного и того же исходного дерева независимо строят каждый свой собственный набор новых объектов — они не пишут ни в какое общее состояние, кроме, возможно, собственных внутренних полей самого экземпляра `ExpressionVisitor` (например, `Members` в `MemberCollector` из вопроса 55 — но это поле принадлежит конкретному **экземпляру** визитора; если каждый поток создаёт свой отдельный экземпляр `MemberCollector`, конфликтов нет). Проблема возникла бы, только если **один и тот же экземпляр** визитора с изменяемым внутренним состоянием переиспользуется одновременно в нескольких потоках — это уже не про сами expression trees, а про обычное правило «нестатичные mutable-объекты не потокобезопасны, если не спроектированы специально».

**Ресурсы:** [API: ExpressionVisitor](https://learn.microsoft.com/dotnet/api/system.linq.expressions.expressionvisitor)

---

### 195. [I] Почему статический кэш скомпилированных делегатов (`ConcurrentDictionary<TKey, Delegate>`) — правильный выбор структуры данных, а не обычный `Dictionary`?

**Ответ.** Обычный `Dictionary<TKey,TValue>` не потокобезопасен для конкурентной записи (и даже для одновременного чтения во время записи из другого потока) — при высоконагруженном веб-приложении, где кэш компилированных предикатов/делегатов заполняется «лениво» при первом обращении с разных потоков одновременно (например, для разных `cacheKey`, вопрос 128), использование обычного `Dictionary` без внешней синхронизации привело бы к повреждению внутренней структуры словаря и непредсказуемым исключениям. `ConcurrentDictionary` спроектирован именно для этого паттерна использования (`GetOrAdd`), обеспечивая корректность при конкурентном доступе без необходимости вручную оборачивать каждую операцию в `lock`.

**Ресурсы:** [API: ConcurrentDictionary<TKey,TValue>](https://learn.microsoft.com/dotnet/api/system.collections.concurrent.concurrentdictionary-2) · [API: ConcurrentDictionary.GetOrAdd](https://learn.microsoft.com/dotnet/api/system.collections.concurrent.concurrentdictionary-2.getoradd)

---
## Группа 23: Чек-лист лучших практик

### 196. [I] Чек-лист: когда стоит использовать `Expression<TDelegate>` вместо простого `Func<TDelegate>` в публичном API библиотеки?

**Ответ.** Используйте `Expression<TDelegate>`, если вызывающему коду (или вашей же реализации) нужно **не только выполнить**, но и **проанализировать/транслировать/изменить** переданную логику: (1) API транслирует запрос в другой язык (SQL, HTTP-фильтр) — как `IQueryable`; (2) API нужно извлечь метаданные из выражения — имя свойства, вызываемый метод — как FluentValidation/HTML-хелперы (вопросы 140-141); (3) API комбинирует несколько выражений в одно составное дерево (спецификации, `PredicateBuilder`). Используйте обычный `Func`/`Action`, если логика **только выполняется** — дополнительная сложность и накладные расходы построения/компиляции дерева не окупаются, если анализ выражения никогда не требуется (лишний `Compile()` на каждый вызов — частая, легко избежимая ошибка производительности, см. вопрос 65).

**Ресурсы:** [MS Docs: Expression Trees Overview](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/)

---

### 197. [I] Чек-лист: пять типичных ошибок при работе с Expression Trees, которые стоит проверять на code review.

**Ответ.**
1. **`.Compile()` внутри цикла/на каждый запрос** вместо кэширования делегата (вопрос 65, 132).
2. **Два независимо построенных `Expression<Func<T,bool>>` объединяются `Expression.AndAlso` без унификации параметров** — приведёт к `InvalidOperationException` о несвязанной переменной (вопрос 91).
3. **Захват `IDisposable`-ресурса** в выражении, чей делегат может быть вызван после освобождения ресурса (вопрос 118).
4. **Динамическое построение предикатов по строковому имени поля без валидации** — риск `ArgumentException` в рантайме или, при использовании парсеров строк, риск некорректного/небезопасного доступа (вопрос 97, 102, 180).
5. **Использование `InvocationExpression`/динамически скомбинированных выражений с EF Core без `.AsExpandable()`/явного разворачивания** — `NotSupportedException: could not be translated` (вопрос 93).

**Ресурсы:** [MS Docs: Execute expression trees — Caveats](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/expression-trees-execution#caveats)

---

### 198. [A] Чек-лист: как спроектировать публичный API, принимающий `Expression<Func<T,bool>>` от вызывающего кода, устойчивым к будущим изменениям?

**Ответ.**
1. Не полагайтесь на то, что дерево будет **полностью** транслируемо во что угодно — документируйте (и по возможности проверяйте на этапе построения запроса, а не только в рантайме БД) какой поднабор конструкций поддерживается.
2. Явно бросайте информативные исключения для неподдерживаемых узлов (вопрос 187), а не «молчаливо» деградируйте до неверного поведения.
3. Если API будет обёрнут в комбинаторы (`And`/`Or`) — обеспечьте корректную обработку унификации параметров «из коробки» (не заставляйте каждого потребителя API самостоятельно решать проблему из вопроса 91).
4. Учитывайте, что новые версии C# могут расширить/ограничить набор конструкций, представимых в дереве (вопрос 106-115) — не привязывайтесь жёстко к текущему списку типов узлов без запасного `default`/`NotSupportedException`-пути в собственных визиторах.

**Ресурсы:** [MS Docs: Expression Trees — Limitations](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/#limitations)

---

### 199. [I] Чек-лист: пять вопросов, которые стоит задать себе перед тем как писать код, динамически строящий Expression Trees вручную, вместо использования готовой библиотеки.

**Ответ.**
1. Решает ли задачу System.Linq.Dynamic.Core (строковые фильтры/сортировки, вопрос 100) — не изобретаю ли я то же самое хуже и без тестового покрытия?
2. Нужна ли действительно динамическая структура (число условий неизвестно заранее), или это конечный, известный на этапе разработки набор комбинаций, которые проще выразить обычными C#-лямбдами напрямую?
3. Есть ли план кэширования построенных деревьев/скомпилированных делегатов, или каждый вызов будет заново строить и компилировать (вопрос 65, 130)?
4. Как будет валидироваться пользовательский ввод, участвующий в построении дерева (имена полей, операторы) — есть ли белый список (вопрос 97)?
5. Будет ли итоговое дерево передаваться `IQueryable`-провайдеру (нужно избегать неподдерживаемых конструкций, `InvocationExpression` и т.д., вопрос 93) или выполняться только в памяти (ограничений меньше)?

**Ресурсы:** [System.Linq.Dynamic.Core](https://github.com/zzzprojects/System.Linq.Dynamic.Core) · [MS Docs: Build dynamic queries](https://learn.microsoft.com/dotnet/csharp/linq/how-to-build-dynamic-queries)

---

### 200. [G] Итоговый чек-лист для guru-уровня: какие пять концепций отличают уверенное «знание синтаксиса» Expression Trees от глубокого понимания темы?

**Ответ.**
1. **Понимание разницы между построением, компиляцией и выполнением** как трёх раздельных, независимо оптимизируемых этапов (группы 1, 6) — вместо смешанного представления «expression tree просто как способ писать LINQ».
2. **Понимание модели неизменяемости и её следствий** — почему трансформация требует создания нового дерева, почему параметры связываются по ссылке, а не по имени, почему структурное сравнение нетривиально (группы 1, 5, 13).
3. **Понимание границы между тем, что представимо в дереве, и тем, что нет**, и *почему* именно так (стабильность API, отсутствие новых типов узлов) — а не просто запоминание списка ограничений (группа 11).
4. **Понимание того, как провайдеры (EF Core, Moq, AutoMapper) используют дерево именно как данные для анализа**, а не как «более медленный способ выполнить код» — ключевая ментальная модель, объясняющая *зачем* вообще существует эта технология (группы 8, 15).
5. **Практическое умение написать и отладить собственный `ExpressionVisitor`** для реальной задачи — трансформации, комбинирования, трансляции — а не только использовать готовые библиотеки (группы 5, 9, 21).

**Ресурсы:** [MS Docs: Expression Trees — полная серия статей](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/) · [Matt Warren: Building an IQueryable Provider](https://learn.microsoft.com/en-us/archive/blogs/mattwar/linq-building-an-iqueryable-provider-series)

---

## Заключение

Эта подборка охватывает путь от базового различия `Func` vs `Expression<Func<...>>` до написания собственного мини LINQ-провайдера. Для дальнейшей практики рекомендуется: (1) реализовать несколько `ExpressionVisitor`-трансформаций самостоятельно (замена операторов, подстановка параметров, сбор метаданных); (2) пройти серию блог-постов Matt Warren целиком, реализуя код параллельно с чтением; (3) изучить исходный код `Microsoft.EntityFrameworkCore.Query` (открытый на GitHub) — это лучшая production-референс того, как построена промышленная трансляция expression trees в SQL; (4) поэкспериментировать с `System.Linq.Dynamic.Core` для динамических сценариев, прежде чем писать собственный парсер.

**Итоговый список общих источников** — см. раздел «Общие источники» в начале документа.
