# AZONE LANG

## Specification 0.7 — VM Execution Model

**Статус:** Draft  
**Версия:** 0.7  
**Реализация:** C++20  
**Целевая платформа разработки:** Arch Linux  
**Виртуальная машина:** Stack-based VM  
**Формат байткода:** `.azb`  
**Исходный код:** `.az`  
**Расширения компилятора:** `.azm`

---

# 1. Назначение

Specification 0.7 определяет архитектуру исполнения AZONE LANG Virtual Machine.

Спецификация описывает:

- жизненный цикл VM;
- загрузку и исполнение `.azb`;
- внутреннее состояние VM;
- Instruction Pointer;
- Operand Stack;
- Call Stack;
- Stack Frames;
- Local Variables;
- Heap;
- Global Storage;
- Constant Pool;
- Type Metadata;
- создание и уничтожение объектов;
- выполнение методов;
- виртуальные и интерфейсные вызовы;
- исключения;
- Garbage Collector;
- потоки исполнения;
- синхронизацию;
- native calls;
- embedding VM в C++-приложения;
- Hosted и Freestanding режимы.

Цель — определить архитектуру, достаточную для создания первой работающей виртуальной машины на C++20 без сторонних библиотек и фреймворков.

---

# 2. Основные принципы

AZONE VM должна придерживаться следующих принципов:

1. Предсказуемая модель исполнения.
2. Проверка корректности байткода перед запуском.
3. Разделение виртуальной памяти и физической памяти.
4. Разделение managed и native memory.
5. Явное управление состоянием VM.
6. Поддержка объектной модели AZONE LANG.
7. Возможность исполнения нескольких независимых программ.
8. Возможность использования VM внутри других приложений.
9. Возможность отключения hosted runtime для специальных системных конфигураций.
10. Возможность дальнейшего добавления JIT/AOT без изменения семантики языка.

VM является реализацией исполнения AZONE LANG, а не частью самой операционной системы.

---

# 3. Общая архитектура VM

```
AZONE VM
│
├── VM Instance
│   ├── VM State
│   ├── Configuration
│   └── Runtime Context
│
├── Loader
│   ├── AZB Reader
│   ├── Module Loader
│   └── Bytecode Validator
│
├── Execution Engine
│   ├── Interpreter
│   ├── Instruction Decoder
│   └── Instruction Dispatcher
│
├── Execution State
│   ├── Instruction Pointer
│   ├── Operand Stack
│   ├── Call Stack
│   └── Local Variables
│
├── Memory System
│   ├── Managed Heap
│   ├── Native Allocator
│   ├── Global Storage
│   └── GC
│
├── Type System
│   ├── Type Metadata
│   ├── Class Metadata
│   ├── Method Tables
│   └── Interface Tables
│
├── Runtime
│   ├── Exceptions
│   ├── Object Allocation
│   ├── Strings
│   └── Arrays
│
└── Native Bridge
    ├── Native Functions
    ├── FFI
    └── ABI Adapter
```

---

# 4. VM Instance

**VM Instance** — отдельный экземпляр виртуальной машины.

Каждый экземпляр содержит собственное состояние исполнения и ресурсы.

Концептуальная структура:

```c++
class VM
{
public:
    VM();
    ~VM();

    bool initialize();
    bool load(const char* path);
    bool execute();
    void requestShutdown();
    void shutdown();
};
```

Это предварительный интерфейс. Точная сигнатура публичного C++ API может быть уточнена при реализации.

Один процесс хоста может создавать несколько VM Instances.

Например:

```
Host Application
│
├── VM Instance A
├── VM Instance B
└── VM Instance C
```

Каждый экземпляр должен иметь независимые execution state и managed heap, если явно не используется разделяемая инфраструктура.

---

# 5. Жизненный цикл VM

Жизненный цикл VM:

```
Created
   ↓
Initializing
   ↓
Ready
   ↓
Loading
   ↓
Loaded
   ↓
Running
   ↓
Stopped
   ↓
Shutdown
```

Возможны дополнительные состояния:

```
Error
Suspended
Terminating
```

## 5.1 Created

Объект VM создан, но внутренние подсистемы ещё не готовы.

## 5.2 Initializing

Инициализируются allocator, runtime structures и execution engine.

## 5.3 Ready

VM готова загружать модули.

## 5.4 Loading

Загружается и проверяется `.azb`.

## 5.5 Loaded

Программа загружена, зависимости разрешены, байткод проверен.

## 5.6 Running

VM исполняет программу.

## 5.7 Stopped

Исполнение завершилось либо было остановлено.

## 5.8 Shutdown

VM освобождает ресурсы и завершает работу.

Переходы между состояниями должны контролироваться самой VM.

---

# 6. Режимы исполнения

AZONE VM предусматривает два основных режима.

## 6.1 Hosted Mode

Hosted Mode предназначен для работы внутри полноценной операционной системы.

Первая реализация ориентируется на Linux.

Доступны:

- стандартный runtime;
- файловые операции через разрешённый API;
- native calls;
- системные потоки;
- полноценный managed heap;
- GC;
- обработка исключений;
- диагностические сообщения.

Hosted Mode не должен автоматически предоставлять программе неограниченный доступ к возможностям операционной системы.

## 6.2 Freestanding Mode

Freestanding Mode предназначен для ограниченных сред, где полноценный hosted runtime недоступен.

Примеры:

- системные компоненты;
- ранняя инициализация runtime;
- специальные среды исполнения;
- будущие системные компоненты AZONE OS.

В этом режиме могут отсутствовать:

- стандартный файловый API;
- стандартная консоль;
- автоматическое создание потоков;
- некоторые runtime services;
- автоматическая инициализация полноценного GC.

Конкретный набор возможностей задаётся конфигурацией.

**Важно:** Freestanding Mode не делает обычную VM автоматически пригодной для исполнения внутри ядра ОС. Для этого отдельно потребуются соответствующие backend, ABI, ограничения памяти и модель привилегий.

---

# 7. Основные подсистемы исполнения

VM состоит из следующих обязательных подсистем:

|Подсистема|Назначение|
|---|---|
|Loader|Загрузка `.azb`|
|Validator|Проверка структуры и байткода|
|Interpreter|Исполнение инструкций|
|Operand Stack|Временные значения|
|Call Stack|Контекст вызовов|
|Heap|Управляемые объекты|
|Runtime|Объектная модель и стандартные операции|
|Type System|Информация о типах|
|Exception Engine|Обработка исключений|
|GC|Управление временем жизни managed-объектов|
|Native Bridge|Вызов нативных функций|

---

# 8. Execution Context

Каждый исполняющий поток имеет собственный Execution Context.

Он включает:

```
Instruction Pointer
Operand Stack
Call Stack
Current Function
Current Module
Thread State
Exception State
```

Концептуальная структура:

```
struct ExecutionContext
{
    // Current execution state.
    // Exact internal representation is implementation-defined.
};
```

Внутренние указатели VM не должны быть доступны обычному коду AZONE LANG напрямую.

---

# 9. Instruction Pointer

Instruction Pointer, или `IP`, указывает на текущую инструкцию байткода.

Концептуально:

```
IP → Current Opcode
```

После исполнения инструкции IP перемещается к следующей инструкции или изменяется инструкцией перехода.

Например:

```
0000 PUSH_CONST 0
0005 STORE_LOCAL 0
0007 LOAD_LOCAL 0
0009 RETURN
```

IP последовательно проходит эти позиции.

Переходы должны проверяться на допустимость и не должны приводить к исполнению данных за пределами секции Bytecode.

---

# 10. Instruction Dispatcher

Interpreter использует Instruction Dispatcher для определения операции по opcode.

Концептуально:

```
while (context.isRunning())
{
    Opcode opcode = fetchOpcode(context);
    executeInstruction(context, opcode);
}
```

Это схема, а не обязательная реализация.

Первая VM должна использовать простой интерпретатор, понятный для отладки.

Возможные реализации dispatcher:

- `switch` по opcode;
- таблица обработчиков;
- computed goto, если поддерживается выбранным компилятором.

Для переносимой реализации C++20 рекомендуется начать с `switch`.

---

# 11. Operand Stack

Operand Stack используется для промежуточных значений.

Пример:

```
PUSH 10
PUSH 20
ADD
RETURN
```

Состояние:

```
[]
[10]
[10, 20]
[30]
[]
```

Operand Stack является частью Execution Context и принадлежит конкретному потоку исполнения.

---

# 12. Представление значений

VM должна иметь внутреннее представление значения.

Концептуально:

```
struct Value
{
    // Value type tag
    // Payload
};
```

Value может представлять:

- `bool`;
- целочисленные типы;
- числа с плавающей точкой;
- `null`;
- managed references;
- native pointers в разрешённых контекстах;
- значения value types;
- внутренние runtime values.

Конкретная структура должна быть определена так, чтобы не создавать неоправданных накладных расходов для распространённых типов.

---

# 13. Типизированные значения

VM должна различать типы значений.

Например:

```
int32
int64
float32
float64
bool
reference<T>
pointer<T>
```

Одинаковое бинарное представление не означает одинаковую семантику типов.

Операции должны соблюдать правила [[Specification 0.3]].

Нельзя автоматически интерпретировать managed reference как произвольный native pointer.

---

# 14. Stack Underflow и Overflow

VM должна обнаруживать некорректное состояние Operand Stack.

## Stack Underflow

Возникает, когда инструкция пытается извлечь больше значений, чем находится в стеке.

Пример:

```
ADD
```

при пустом стеке.

## Stack Overflow

Возникает, когда стек достигает установленного предела.

VM должна поддерживать настраиваемые лимиты, где это возможно.

При ошибке исполнение должно завершаться контролируемым runtime error, а не повреждением памяти процесса.

---

# 15. Call Stack

Call Stack хранит активные вызовы функций.

Каждый вызов создаёт Stack Frame.

Пример:

```
main()
 └── game.run()
      └── player.update()
           └── physics.calculate()
```

Каждый уровень соответствует отдельному frame.

Call Stack не является тем же самым, что Operand Stack.

- Operand Stack хранит временные операнды.
- Call Stack хранит контексты вызовов.

Физически эти структуры могут быть реализованы раздельно или как единая область с логическим разделением.

Для MVP рекомендуется логически разделить их.

---

# 16. Stack Frame

Каждая активная функция имеет Stack Frame.

Frame содержит:

```
Function ID
Instruction Pointer / Return Position
Local Variables
Operand Stack Base
Caller Frame
Exception State
```

Концептуальная структура:

```
struct StackFrame
{
    // Current function
    // Return position
    // Local storage
    // Operand stack boundary
};
```

Frame создаётся при вызове функции и удаляется после её завершения.

---

# 17. Создание Stack Frame

При `CALL` VM должна:

1. проверить корректность Function ID;
2. проверить количество аргументов;
3. проверить типы аргументов в соответствии с правилами VM;
4. сохранить позицию возврата;
5. создать новый frame;
6. разместить аргументы в локальных слотах;
7. инициализировать остальные локальные переменные;
8. передать управление вызываемой функции.

Точная последовательность внутренних операций определяется реализацией, но наблюдаемое поведение должно соответствовать этим правилам.

---

# 18. Аргументы функций

Для функции:

```AZONE
int32 add(int32 a, int32 b)
{
    return a + b;
}
```

VM может разместить аргументы:

```
local[0] = a
local[1] = b
```

Остальные локальные переменные размещаются в дополнительных слотах.

Индексы локальных слотов должны быть определены компилятором и проверены валидатором.

---

# 19. Возврат из функции

Инструкция `RETURN` завершает текущую функцию.

При возврате VM должна:

1. получить результат, если функция его возвращает;
2. завершить текущий frame;
3. восстановить вызывающий контекст;
4. восстановить позицию возврата;
5. передать результат вызывающей функции.

Для `void`-функций значение результата отсутствует.

---

# 20. Рекурсия

VM должна поддерживать рекурсивные вызовы.

Пример:

```AZONE
int32 factorial(int32 n)
{
    if (n <= 1)
        return 1;

    return n * factorial(n - 1);
}
```

Каждый рекурсивный вызов получает собственный Stack Frame и собственные локальные переменные.

При превышении допустимой глубины вызовов VM должна сообщать контролируемую ошибку.

---

# 21. Heap

Heap предназначен для объектов и других динамически размещаемых данных.

Например:

```AZONE
Player player = new Player();
```

Объект `Player` обычно размещается в managed heap.

Heap не должен смешиваться с Operand Stack или Call Stack.

---

# 22. Managed Heap

Managed Heap обслуживает объекты, жизненным циклом которых управляет runtime.

Он отвечает за:

- выделение памяти;
- регистрацию объектов;
- обнаружение достижимых объектов;
- взаимодействие с GC;
- обработку allocation failure.

Алгоритм GC будет определён в отдельной спецификации памяти и runtime.

---

# 23. Native Memory

Native Memory предназначена для данных, которыми управляет нативный код или явно вызываемые memory API.

Она может использоваться для:

- системных структур;
- C ABI;
- буферов ввода-вывода;
- взаимодействия с библиотеками;
- специальных низкоуровневых задач.

Native Memory не должна автоматически считаться частью managed heap.

---

# 24. Разделение managed и native references

VM должна различать:

```
Managed Reference
Native Pointer
Address
```

Managed Reference используется для обращения к объектам runtime.

Native Pointer предназначен для работы с нативной памятью в разрешённых контекстах.

Address представляет адрес согласно правилам языка и целевой платформы.

Преобразование между ними должно быть явным и ограничиваться соответствующими правилами безопасности.

---

# 25. Global Storage

Global Storage содержит глобальные и статические данные программы.

Например:

```AZONE
static int32 counter = 0;
```

Глобальные значения должны быть доступны в соответствии с правилами видимости и module system.

Инициализация глобальных данных выполняется до использования соответствующего модуля.

Порядок инициализации зависимых модулей должен соответствовать dependency graph.

---

# 26. Constant Pool

Constant Pool содержит неизменяемые константы `.azb`.

VM должна обеспечивать корректный доступ к:

- числовым константам;
- строкам;
- Type IDs;
- Function IDs;
- Field IDs;
- Method IDs.

Индексы Constant Pool должны проверяться.

Попытка доступа к несуществующей константе является ошибкой байткода.

---

# 27. Type Metadata

Type Metadata описывает runtime-типы.

Для класса могут храниться:

```
Type ID
Type Name
Base Type
Implemented Interfaces
Field Metadata
Method Metadata
Object Layout Metadata
GC Metadata
```

Metadata используется для:

- allocation;
- field access;
- virtual dispatch;
- interface dispatch;
- type checks;
- cast;
- reflection в будущем;
- работы GC.

---

# 28. Object Allocation

Инструкция `NEW` создаёт объект.

Общий процесс:

```
NEW Type
   ↓
Validate Type
   ↓
Calculate Allocation Size
   ↓
Allocate Memory
   ↓
Initialize Object Header
   ↓
Initialize Fields
   ↓
Invoke Constructor
   ↓
Return Managed Reference
```

Если выделение памяти невозможно, VM должна обработать allocation failure.

Объект не должен становиться доступным пользовательскому коду до завершения необходимых этапов инициализации.

---

# 29. Конструкторы

Конструкторы вызываются согласно объектной модели [[Specification 0.4]].

VM должна учитывать:

- базовый класс;
- инициализацию полей;
- конструктор текущего класса;
- перегрузку конструкторов;
- ошибки, возникшие во время инициализации.

Если конструктор выбрасывает исключение, объект не должен считаться успешно созданным.

Точный механизм очистки частично инициализированного объекта определяется Runtime.

---

# 30. Виртуальные вызовы

Для `CALL_VIRTUAL` VM должна:

1. получить объект;
2. определить его фактический тип;
3. найти соответствующий виртуальный слот;
4. определить реализацию;
5. создать Stack Frame;
6. передать управление методу.

Если объект равен `null`, должна возникнуть ошибка null reference.

---

# 31. Интерфейсные вызовы

Для `CALL_INTERFACE` VM должна определить реализацию метода через metadata интерфейсов.

Проверяется:

- наличие объекта;
- наличие соответствующей реализации;
- корректность сигнатуры;
- доступность метода.

Отсутствующая реализация считается ошибкой runtime metadata или байткода.

---

# 32. Статические вызовы

Статические методы вызываются без `this`.

VM не должна создавать фиктивную managed reference для статического метода.

Перед вызовом должны быть проверены:

- Function ID;
- количество аргументов;
- типы аргументов;
- инициализация типа, если она требуется.

---

# 33. Свойства

Свойства из [[Specification 0.4]] могут быть представлены вызовами методов:

```
get_Property
set_Property
```

Автоматические свойства могут использовать скрытые поля.

Таким образом, VM не обязана иметь отдельные инструкции для каждого свойства.

---

# 34. Null Reference

VM должна корректно обрабатывать `null`.

Операции, требующие существующего объекта, не должны разыменовывать `null`.

К ним относятся:

- field access;
- instance method call;
- virtual call;
- interface call;
- некоторые операции преобразования типов.

Ошибка должна быть представлена контролируемым runtime exception или эквивалентной диагностикой.

---

# 35. Arrays

Массивы являются runtime-объектами, если типовая модель не определяет специального value representation.

VM должна поддерживать:

- создание массива;
- определение длины;
- чтение элемента;
- запись элемента;
- проверку границ;
- проверку совместимости типов;
- взаимодействие с GC.

Для обычного безопасного режима доступ по индексу проверяется.

---

# 36. Strings

Строки представляются в UTF-8 согласно [[Specification 0.5]].

VM должна обеспечивать:

- создание строковых значений;
- хранение строковых констант;
- передачу строк между функциями;
- сравнение строк;
- корректное взаимодействие с native API.

Строковая константа может храниться в Constant Pool или String Table.

Runtime должен отличать managed string от C-строки в native memory.

---

# 37. Exception Engine

Exception Engine отвечает за обработку исключений.

Минимальные возможности:

```
THROW
Catch Handler
Stack Unwinding
Exception Type Check
Unhandled Exception Reporting
```

---

# 38. Обработка исключения

При выполнении `THROW`:

1. VM получает объект или значение исключения;
2. определяет текущую позицию исполнения;
3. ищет подходящий handler;
4. проверяет совместимость типа исключения;
5. при необходимости удаляет текущие frames;
6. передаёт управление обработчику;
7. если обработчик не найден — продолжает unwinding.

Если исключение не обработано, VM должна завершить текущую программу контролируемым образом.

---

# 39. Stack Unwinding

Во время unwinding VM должна поддерживать корректное состояние:

- Call Stack;
- локальных переменных;
- временных значений;
- GC roots;
- ресурсов, требующих гарантированной очистки.

Правила детерминированной очистки ресурсов должны быть определены языком и runtime.

Нельзя полагаться на момент срабатывания GC для закрытия файлов, освобождения блокировок или выполнения других критических операций.

---

# 40. Garbage Collector

GC управляет временем жизни managed objects.

VM должна предоставлять GC возможность:

- обнаруживать корни;
- обходить ссылки;
- определять достижимые объекты;
- освобождать недостижимые объекты;
- взаимодействовать с execution contexts;
- учитывать native references, если они зарегистрированы.

Алгоритм GC пока не фиксируется окончательно.

Для MVP рекомендуется начать с простого non-moving mark-and-sweep collector.

Такой вариант проще реализовать и отладить, чем перемещающий GC.

---

# 41. GC Roots

К GC roots относятся:

- ссылки в локальных переменных;
- ссылки в Operand Stack;
- ссылки в активных Stack Frames;
- глобальные ссылки;
- статические поля;
- runtime handles;
- зарегистрированные native roots;
- другие корни, определённые runtime.
    

GC не должен освобождать объект, на который существует действующая managed reference.

---

# 42. Safepoints

Safepoint — точка, в которой VM может безопасно приостановить исполнение для runtime-операций.

Например:

```
Before allocation
Before native call
At selected loop backedges
At function calls
```

GC может запрашивать safepoint для выполнения сборки мусора.

Конкретный набор safepoints определяется реализацией VM и metadata `.azb`.

---

# 43. Native Handles

Native code может временно хранить ссылку на managed object.

В таком случае обычного локального указателя недостаточно, если GC не отслеживает эту ссылку.

Поэтому предусматривается механизм:

```
GC Handle
```

Handle регистрирует ссылку для GC и обеспечивает корректность её времени жизни.

Использование native references должно соответствовать правилам Native API.

---

# 44. Потоки исполнения

AZONE VM должна поддерживать концепцию VM Thread.

Каждый VM Thread имеет собственный:

```
Execution Context
Operand Stack
Call Stack
Instruction Pointer
Thread State
```

Managed Heap и некоторые runtime metadata могут быть общими для потоков одного VM Instance.

---

# 45. VM Thread vs OS Thread

VM Thread и OS Thread — разные понятия.

В MVP рекомендуется использовать модель:

```
One VM Thread
        ↕
One OS Thread
```

Это упрощает реализацию и интеграцию с native code.

В будущем может появиться поддержка:

- green threads;
- fibers;
- cooperative scheduling;
- user-space scheduling.

Эти возможности не обязательны для первой VM.

---

# 46. Thread States

Предусматриваются состояния:

```
Created
Runnable
Running
Blocked
Suspended
Terminating
Terminated
```

Переходы между состояниями управляются VM и её thread subsystem.

---

# 47. Синхронизация

VM должна иметь архитектурную возможность поддерживать:

- mutex;
- condition variable;
- atomic operations;
- thread join;
- thread state transitions;
- synchronization с native code.

Часть механизмов может предоставляться стандартной библиотекой, а часть — native runtime.

Точные API будут определены в будущих спецификациях concurrency.

---

# 48. Shared Objects

Если объект доступен нескольким потокам, программа должна соблюдать правила синхронизации языка.

VM не должна автоматически считать, что все операции с полями атомарны.

В частности:

```
counter = counter + 1;
```

не гарантирует атомарность.

Для атомарных операций предусматривается `atomic<T>` согласно [[Specification 0.5]].

---

# 49. Memory Visibility

Для многопоточных программ VM должна соблюдать модель памяти AZONE LANG.

Вызовы native functions и synchronization primitives должны корректно взаимодействовать с моделью памяти языка.

`volatile` не заменяет атомарные операции или синхронизацию.

---

# 50. Native Bridge

Native Bridge позволяет VM вызывать функции, реализованные вне AZONE LANG.

Схема:

```
AZONE Bytecode
      ↓
VM Interpreter
      ↓
Native Bridge
      ↓
ABI Adapter
      ↓
Native Function
```

Native Bridge отвечает за:

- разрешение символа;
- проверку сигнатуры;
- преобразование аргументов;
- вызов функции;
- преобразование результата;
- обработку ошибок;
- взаимодействие с GC;
- соблюдение ABI.

---

# 51. Native Function Registry

VM должна иметь registry нативных функций.

Концептуально:

```
Symbol Name
Function Signature
Calling Convention
Target ABI
Access Policy
Native Function Address
```

Регистрация должна происходить через контролируемый API.

Непроверенный `.azb` не должен иметь возможность произвольно вызывать любой адрес памяти как функцию.

---

# 52. Native Calls и исключения

Исключения AZONE LANG не должны бесконтрольно пересекать границу C ABI.

Native Bridge должен преобразовывать ошибки нативного вызова в согласованный runtime error или exception.

Аналогично native code не должен произвольно передавать исключение C++ через внутренние VM frames.

---

# 53. Embedding VM

AZONE VM должна поддерживать встраивание в C++-программы.

Примеры:

- приложения;
- редакторы;
- игровые движки;
- инструменты разработчика;
- сервисы;
- хосты для скриптовых расширений.

Предварительный интерфейс:

```
class VM
{
public:
    bool initialize();
    bool loadModule(const char* path);
    bool call(const char* functionName);
    void requestShutdown();
    void shutdown();
};
```

Окончательный API должен поддерживать передачу аргументов, результатов и диагностик без обязательного использования глобального состояния.

---

# 54. Несколько VM Instances

Приложение-хост может создавать несколько экземпляров VM.

Каждый экземпляр должен иметь независимые:

- execution state;
- module state;
- global storage;
- managed heap;
- runtime configuration.

Совместное использование модулей или immutable metadata может быть оптимизацией в будущем, но не является обязательным для MVP.

---

# 55. Управление ошибками VM API

Публичный API должен сообщать об ошибках явно.

Например:

```
InitializationFailed
ModuleLoadFailed
InvalidBytecode
EntryPointNotFound
ExecutionFailed
NativeCallFailed
OutOfMemory
ShutdownFailed
```

Конкретная модель представления ошибок — enum, result type или error object — определяется C++ API.

---

# 56. Ограничения ресурсов

VM должна иметь возможность задавать ограничения:

```
Maximum Operand Stack
Maximum Call Stack Depth
Maximum Heap Size
Maximum Loaded Modules
Maximum Threads
Execution Budget
```

Не все лимиты обязательны для Freestanding Mode.

Лимиты должны быть проверяемыми, чтобы ошибки не приводили к неконтролируемому повреждению памяти.

---

# 57. Execution Budget

Для встраиваемой VM предусматривается возможность ограничивать объём исполнения.

Например:

```
Maximum Instruction Count
Maximum Execution Time
Cancellation Request
```

Временной лимит может быть приблизительным, поскольку отдельная native function способна задержать выполнение.

VM должна проверять запросы остановки в безопасных точках исполнения.

---

# 58. VM Shutdown

При завершении VM должна:

1. прекратить запуск новых задач;
2. остановить или корректно завершить VM Threads;
3. завершить выполняемую программу согласно выбранной политике;
4. освободить runtime handles;
5. завершить активные runtime services;
6. освободить managed heap;
7. освободить загруженные ресурсы;
8. перевести экземпляр в состояние `Shutdown`.

Деструктор C++-объекта VM должен выполнять безопасную очистку уже инициализированных ресурсов.

---

# 59. Runtime Diagnostics

VM должна предоставлять диагностическую информацию:

```
Error Type
Error Message
Module
Function
Bytecode Offset
Source File
Source Line
Stack Trace
```

Если `.azb` не содержит debug information, VM всё равно должна уметь сообщить как минимум Function ID и bytecode offset, когда эти сведения доступны.

---

# 60. VM Logging

Для диагностики могут использоваться уровни:

```
ERROR
WARN
INFO
DEBUG
TRACE
```

Встроенный logger не обязан быть сложным.

В Freestanding Mode вывод может предоставляться через внешний callback.

---

# 61. Hosted Runtime

Hosted Runtime предоставляет возможности, зависящие от хостовой среды:

- файловый ввод-вывод;
- стандартный вывод;
- системные потоки;
- native libraries;
- таймеры;
- базовые системные функции.

Конкретный API должен быть определён отдельной спецификацией стандартной библиотеки.

VM не должна смешивать функции hosted runtime с обязательными возможностями самого interpreter.

---

# 62. Freestanding Runtime

Freestanding Runtime должен минимизировать зависимости от хостовой операционной системы.

Его конфигурация может требовать, чтобы хост предоставлял:

```
Memory Allocator
Output Callback
Native Function Registry
Execution Entry Point
Optional Scheduler Hooks
```

Эта модель позволяет встраивать runtime в специализированные среды.

Однако работа без Linux userspace не означает автоматически поддержку kernel-mode исполнения. Для такого использования требуются дополнительные архитектурные решения.

---

# 63. Применение в AZONE OS

AZONE LANG предназначен для использования в будущей AZONE OS на базе Linux kernel.

Потенциальные системные компоненты:

```
Process Services
System Services
Memory Management Services
Filesystem Services
Network Services
Display Server
GUI Services
Applications
```

VM может служить runtime для user-space компонентов.

При этом необходимо разделять:

1. обычные приложения AZONE LANG;
2. системные службы AZONE OS;
3. компоненты, взаимодействующие с Linux kernel;
4. код, исполняющийся в privileged context.

Эти уровни имеют разные требования к безопасности, памяти и ABI.

---

# 64. VM и процессы операционной системы

В первой реализации один VM Instance может исполняться внутри одного процесса Linux.

Например:

```
Linux Process
 └── AZONE VM
      ├── VM Thread A
      └── VM Thread B
```

Это не означает, что VM Thread автоматически является отдельным процессом операционной системы.

Будущая модель процессов AZONE OS должна отдельно определить:

- процесс;
- адресное пространство;
- права доступа;
- IPC;
- системные вызовы;
- загрузку исполняемых файлов;
- изоляцию процессов.

---

# 65. Изоляция

По умолчанию отдельные VM Instances должны иметь независимые managed heaps и execution state.

Но VM Instance сама по себе не является полноценной security sandbox.

Если VM предоставляет native calls, файловый API или другие системные возможности, их полномочия должны контролироваться хостом.

Надёжная изоляция требует ограничений не только на байткод, но и на все доступные внешние API.

---

# 66. Native Memory и GC

GC не должен освобождать native memory, если её жизненным циклом управляет внешний код.

Если native code хранит managed reference, он должен использовать зарегистрированный GC Handle или другой механизм, предусмотренный runtime.

Если managed object содержит native resource, освобождение ресурса должно выполняться через определённый механизм детерминированной очистки.

---

# 67. VM Configuration

VM должна иметь конфигурацию, определяющую доступные функции.

Пример:

```
Execution Mode
Heap Limit
Stack Limit
GC Enabled
Debug Enabled
Native Calls Enabled
Thread Support
Module Search Paths
Diagnostics Callback
```

Фактические параметры могут передаваться через C++ API или конфигурационный объект.

---

# 68. Runtime API и Bytecode API

Необходимо различать:

**Runtime API** — функции, доступные программам AZONE LANG.

**VM API** — интерфейс управления виртуальной машиной со стороны C++ host.

Например:

```
AZONE LANG:
    System.Console.write()

C++ Host:
    vm.initialize()
    vm.loadModule(...)
    vm.execute()
```

Эти интерфейсы могут взаимодействовать, но не являются одним и тем же API.

---

# 69. Порядок реализации MVP

Рекомендуемый порядок разработки:

## Этап 1 — Minimal VM

- создание VM Instance;
- чтение `.azb`;
- проверка заголовка;
- operand stack;
- instruction pointer;
- interpreter;
- `PUSH`;
- `POP`;
- `ADD`;
- `SUB`;
- `MUL`;
- `DIV`;
- `RETURN`.

## Этап 2 — Functions

- Function Table;
- local variables;
- Stack Frames;
- `CALL`;
- рекурсия;
- возврат значений.

## Этап 3 — Program Runtime

- Constant Pool;
- Global Storage;
- загрузка модулей;
- entry point;
- runtime diagnostics.

## Этап 4 — Object Runtime

- Type Metadata;
- классы;
- allocation;
- поля;
- методы;
- виртуальные вызовы;
- интерфейсы.

## Этап 5 — Memory Management

- managed heap;
- GC roots;
- safepoints;
- первый GC;
- native allocation API.

## Этап 6 — Advanced Runtime

- исключения;
- native bridge;
- многопоточность;
- embedding API;
- Freestanding configuration.

---

# 70. Требования к тестированию

VM должна иметь автоматические тесты как минимум для:

- корректной загрузки `.azb`;
- некорректного заголовка;
- некорректных offsets;
- неизвестного opcode;
- stack underflow;
- stack overflow;
- арифметических операций;
- переходов;
- вызовов функций;
- рекурсии;
- локальных переменных;
- создания объектов;
- доступа к полям;
- виртуального dispatch;
- обработки исключений;
- выделения памяти;
- работы GC;
- native calls;
- остановки VM;
- повторного создания VM Instance.

Тесты должны быть воспроизводимыми и не зависеть от случайного состояния предыдущего теста.

---

# 71. Архитектурные ограничения MVP

В первой версии:

- используется interpreter;
- JIT не требуется;
- AOT backend не требуется;
- VM Threads по возможности соответствуют OS Threads;
- GC может быть non-moving;
- виртуальная машина исполняет portable bytecode;
- native calls регистрируются явно;
- полноценная sandbox-система не считается встроенной автоматически;
- kernel-mode execution не входит в обязательный MVP.

Эти ограничения позволяют получить рабочую VM до добавления сложных оптимизаций.

---

# 72. Критерии готовности Specification 0.7

Спецификация считается реализуемой, когда архитектура позволяет ответить на вопросы:

- Как создаётся VM?
- Как загружается `.azb`?
- Как начинается исполнение?
- Где хранится IP?
- Как работает Operand Stack?
- Как создаются Stack Frames?
- Как выполняются функции?
- Как создаются объекты?
- Где хранятся managed objects?
- Как обнаруживаются GC roots?
- Как обрабатываются исключения?
- Как вызываются native functions?
- Как VM останавливается?
- Как она встраивается в C++ host?
- Какие компоненты нужны для Freestanding Mode?

---

# 73. Итоговая архитектура

```
                    C++ Host
                       │
                       ▼
                  VM Instance
                       │
              ┌────────┴────────┐
              ▼                 ▼
            Loader          Runtime Setup
              │                 │
              ▼                 │
          AZB Validator          │
              │                 │
              └────────┬────────┘
                       ▼
                 Execution Engine
                       │
             ┌─────────┼─────────┐
             ▼         ▼         ▼
            IP      Operand    Call Stack
                       Stack      │
                                  ▼
                              Stack Frames
                       │
              ┌────────┴────────┐
              ▼                 ▼
          Managed Heap       Type Metadata
              │                 │
              ▼                 ▼
              GC            Object Runtime
                       │
                       ▼
                  Native Bridge
                       │
                       ▼
                  Host Functions
```

---

# 74. Что фиксирует Specification 0.7

Данный документ определяет:

- жизненный цикл VM;
- VM Instance;
- Hosted и Freestanding режимы;
- Execution Context;
- Instruction Pointer;
- Instruction Dispatcher;
- Operand Stack;
- Call Stack;
- Stack Frames;
- Local Variables;
- Heap;
- Global Storage;
- Constant Pool;
- Type Metadata;
- object allocation;
- method dispatch;
- exception handling;
- GC integration;
- VM Threads;
- synchronization requirements;
- Native Bridge;
- embedding;
- ограничения ресурсов;
- shutdown;
- diagnostics;
- план реализации MVP.

---

# 75. Следующая спецификация

## Specification 0.8 — AZM Compiler Extension System

Следующий этап посвящён `.azm` — расширениям компилятора.

В Specification 0.8 необходимо определить:

- формат `.azm`;
- загрузчик расширений;
- metadata расширения;
- API расширений на C++20;
- регистрацию новых типов;
- регистрацию операторов;
- регистрацию функций;
- добавление ключевых слов;
- расширение AST;
- compiler passes;
- добавление bytecode instructions;
- runtime integration;
- версионирование API;
- совместимость расширений;
- диагностику;
- зависимости;
- безопасное отключение расширений;
- взаимодействие расширений с будущей AZONE OS.

**Главная цель:** сделать AZONE LANG расширяемым, но сохранить предсказуемость компилятора и совместимость уже существующего кода.