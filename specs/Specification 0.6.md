# AZONE LANG

## Specification 0.6 — Bytecode & `.azb` Binary Format

**Статус:** Draft  
**Версия:** 0.6  
**Проект:** AZONE LANG  
**Целевая реализация:** C++20  
**Основная платформа разработки:** Arch Linux  
**VM:** Stack-based Virtual Machine  
**Исходный код:** `.az`  
**Расширения компилятора:** `.azm`  
**Байткод:** `.azb`

---

# 1. Назначение спецификации

Данная спецификация определяет формат скомпилированного файла **`.azb`** и бинарный байткод AZONE LANG.

`.azb` является результатом работы компилятора AZONE LANG и предназначен для загрузки и исполнения виртуальной машиной AZONE LANG.

Основные задачи формата:

- хранение исполняемого байткода;
- хранение информации о типах;
- хранение классов и методов;
- хранение констант;
- хранение строк;
- хранение глобальных данных;
- хранение метаданных модулей;
- хранение информации для сборщика мусора;
- хранение информации об исключениях;
- хранение отладочной информации;
- поддержка native/FFI-вызовов;
- возможность валидации файла до исполнения;
- возможность расширения формата в будущих версиях.

---

# 2. Общая архитектура

Компиляция выполняется по схеме:

```text
.az
 ↓
Lexer
 ↓
Parser
 ↓
AST
 ↓
Semantic Analysis
 ↓
Compiler
 ↓
Bytecode Generator
 ↓
.azb
 ↓
AZONE VM
 ↓
Runtime
```

Для программы с несколькими модулями:

```text
Module A ─┐
Module B ─┼→ Compiler → .azb
Module C ─┘
```

`.azb` может содержать один основной модуль или набор скомпилированных модулей в зависимости от режима компиляции.

---

# 3. Основные требования к `.azb`

Формат `.azb` должен быть:

- бинарным;
- детерминированным;
- проверяемым;
- расширяемым;
- версионируемым
- независимым от конкретной реализации VM настолько, насколько это возможно;
- пригодным для быстрого чтения;
- пригодным для memory mapping в будущем;
- безопасным для обработки потенциально повреждённых файлов.

VM **не должна начинать исполнение `.azb` до завершения его валидации**.

---

# 4. Endianness

Первая версия AZONE LANG использует:

```text
Little Endian
```

Это относится к:

- заголовку;
- индексам;
- размерам;
- смещениям;
- числовым операндам;
- таблицам;
- секциям.

В будущем формат может поддерживать другие варианты endian через поле `Endianness` в заголовке.

---

# 5. Структура `.azb`

Общая структура:

```text
+----------------------+
| Header               |
+----------------------+
| Section Table        |
+----------------------+
| Constant Pool        |
+----------------------+
| String Table         |
+----------------------+
| Type Table           |
+----------------------+
| Function Table       |
+----------------------+
| Class Table          |
+----------------------+
| Field Table          |
+----------------------+
| Method Table         |
+----------------------+
| Module Metadata      |
+----------------------+
| Global Data          |
+----------------------+
| Bytecode             |
+----------------------+
| Exception Data       |
+----------------------+
| GC Metadata          |
+----------------------+
| Relocation Table     |
+----------------------+
| Symbol Table         |
+----------------------+
| Debug Information    |
+----------------------+
| Checksum             |
+----------------------+
```

Не все секции обязаны присутствовать.

Например:

- release-сборка может не содержать Debug Information;
- программа без native-кода может не содержать Relocation Table;
- библиотека без глобальных переменных может не содержать Global Data.

---

# 6. AZB Header

Каждый `.azb` начинается с фиксированного заголовка.

Концептуальная структура:

```AZONE
struct AZBHeader
{
    char     magic[4];

    uint16_t formatMajor;
    uint16_t formatMinor;

    uint16_t languageMajor;
    uint16_t languageMinor;

    uint16_t vmMajor;
    uint16_t vmMinor;

    uint8_t  endianness;
    uint8_t  targetType;
    uint16_t targetArchitecture;

    uint32_t flags;

    uint32_t sectionCount;

    uint64_t sectionTableOffset;

    uint64_t entryPoint;

    uint64_t fileSize;

    uint64_t checksumOffset;
};
```

Фактическое бинарное представление структуры может быть изменено для оптимизации, однако семантика полей должна сохраняться.

---

# 7. Magic Number

Magic Number:

```text
AZB\0
```

В hex:

```text
41 5A 42 00
```

VM обязана проверить Magic Number до обработки остальных полей.

Если magic не совпадает:

```text
Invalid AZB file
```

Файл не загружается.

---

# 8. Версия формата

`.azb` содержит две версии:

```text
formatMajor
formatMinor
```

### Major

Изменение `Major` означает несовместимое изменение бинарного формата.

Пример:

```text
0.x → 1.x
```

может потребовать новой VM.

### Minor

Изменение `Minor` должно по возможности сохранять обратную совместимость.

Пример:

```text
0.6 → 0.7
```

может добавлять новые необязательные секции.

---

# 9. Версия языка

Файл также содержит версию AZONE LANG:

```text
languageMajor
languageMinor
```

Она позволяет VM и инструментам определить, какой версии языка соответствует файл.

Например:

```text
AZONE LANG 0.6
```

---

# 10. Версия VM

Поле:

```text
vmMajor
vmMinor
```

указывает минимальную версию VM, необходимую для исполнения файла.

VM должна проверить совместимость:

```text
AZB VM requirement
        ↓
Current VM
        ↓
Compatible?
```

Если нет:

```text
Incompatible VM version
```

---

# 11. Endianness ID

Используются значения:

```text
0 = Little Endian
1 = Big Endian
```

Для первой реализации:

```text
endianness = 0
```

---

# 12. Target Type

`.azb` различает VM bytecode и native-компоненты.

Предлагаемые значения:

```text
0 = VM Portable
1 = Native
2 = Hybrid
```

### VM Portable

Файл содержит только VM-байткод и runtime metadata.

### Native

Файл предназначен для конкретной архитектуры.

### Hybrid

Файл содержит VM-байткод и native-компоненты.

---

# 13. Target Architecture

Начальная архитектура:

```text
x86-64
```

В будущем:

```text
AArch64
RISC-V64
```

Идентификаторы должны быть определены спецификацией.

Например:

```text
0x0001 = x86-64
0x0002 = AArch64
0x0003 = RISC-V64
```

Для чистого portable VM bytecode архитектура может иметь значение:

```text
ARCH_VM = 0
```

---

# 14. AZB Flags

`flags` является битовой маской.

Предлагаемые флаги:

```text
0x00000001 = HasDebugInfo
0x00000002 = HasExceptions
0x00000004 = HasGCMetadata
0x00000008 = HasRelocations
0x00000010 = HasSymbols
0x00000020 = HasNativeCode
0x00000040 = HasGlobalData
0x00000080 = HasEntryPoint
0x00000100 = IsLibrary
0x00000200 = IsExecutable
0x00000400 = IsModule
0x00000800 = IsSigned
```

Неизвестные флаги должны обрабатываться в соответствии с версией формата.

---

# 15. Section Table

После Header располагается таблица секций.

Каждая секция описывается структурой:

```AZONE
struct AZBSection
{
    uint32_t type;
    uint32_t flags;

    uint64_t offset;
    uint64_t size;

    uint64_t alignment;

    uint32_t checksum;
};
```

Секция должна находиться в пределах файла:

```text
offset + size <= fileSize
```

В противном случае `.azb` считается повреждённым.

---

# 16. Типы секций

Предлагаемые типы:

```text
0x01 Header
0x02 ConstantPool
0x03 StringTable
0x04 TypeTable
0x05 FunctionTable
0x06 ClassTable
0x07 FieldTable
0x08 MethodTable
0x09 ModuleMetadata
0x0A GlobalData
0x0B Bytecode
0x0C ExceptionData
0x0D GCMetadata
0x0E RelocationTable
0x0F SymbolTable
0x10 DebugInfo
0x11 Checksum
```

Будущие версии могут добавлять новые типы.

---

# 17. Constant Pool

Constant Pool содержит значения, используемые байткодом.

Возможные типы:

```text
Integer
UnsignedInteger
Float
Double
StringReference
TypeReference
FunctionReference
FieldReference
MethodReference
Address
Null
```

Пример:

```text
Constant #0 = 42
Constant #1 = "Hello"
Constant #2 = Type::Player
Constant #3 = Function::main
```

---

# 18. Constant Pool Entry

Каждый элемент содержит тип:

```AZONE
struct ConstantEntry
{
    uint8_t type;
    uint32_t size;
    uint64_t data;
};
```

Для более крупных данных допускается хранение ссылки на дополнительную область.

---

# 19. String Table

Все строки `.azb` хранятся в UTF-8.

String Table может содержать:

- имена классов;
- имена методов;
- имена полей;
- имена модулей;
- namespace;
- debug information;
- экспортируемые символы;
- строки программы.

Каждая строка должна иметь:

```text
offset
length
```

Строки не обязаны завершаться `\0`.

---

# 20. Type Table

Type Table содержит metadata типов.

Для каждого типа:

```text
Type ID
Name
Flags
Base Type
Size
Alignment
Field Count
Method Count
GC Information
```

Пример:

```text
Type #12

Name: Player
Base: Entity
Size: 64
Alignment: 8
```

---

# 21. Type IDs

Каждый тип получает уникальный ID внутри `.azb`.

Например:

```text
0 = void
1 = bool
2 = int32
3 = int64
4 = uint32
5 = uint64
6 = float32
7 = float64
...
```

Пользовательские классы получают последующие ID.

---

# 22. Function Table

Function Table содержит функции, включая:

- обычные функции;
- `main`;
- статические методы;
- конструкторы;
- специальные runtime-функции;
- bridge-функции.

Концептуальная структура:

```AZONE
struct AZBFunction
{
    uint32_t name;
    uint32_t returnType;

    uint32_t parameterCount;
    uint32_t localCount;

    uint32_t maxStack;

    uint64_t bytecodeOffset;
    uint64_t bytecodeSize;

    uint32_t flags;
};
```

---

# 23. Local Slots

Локальные переменные функции размещаются в local slots.

Например:

```AZONE
function add(int32 a, int32 b)
{
    int32 result = a + b;
    return result;
}
```

VM может представить их:

```text
local[0] = a
local[1] = b
local[2] = result
```

---

# 24. Operand Stack

Каждая функция также использует operand stack.

Пример:

```text
PUSH 10
PUSH 20
ADD
RETURN
```

Состояние:

```text
[]
[10]
[10, 20]
[30]
[]
```

---

# 25. Bytecode Encoding

Каждая инструкция начинается с opcode.

Базовый формат:

```text
[ opcode ][ operands... ]
```

`opcode`:

```text
uint8
```

Операнды первой версии используют little-endian фиксированные размеры.

Основные размеры:

```text
u8
u16
u32
u64
i32
i64
```

Индексы таблиц в MVP:

```text
uint32
```

Это упрощает реализацию компилятора и VM.

---

# 26. Instruction Pointer

VM содержит:

```text
IP — Instruction Pointer
```

IP указывает на текущую инструкцию в Bytecode Section.

После исполнения:

```text
IP → next instruction
```

Для переходов используются bytecode-relative offsets.

---

# 27. Базовый набор инструкций

Первая версия VM должна поддерживать как минимум:

```text
NOP

PUSH
POP
DUP

LOAD_LOCAL
STORE_LOCAL

LOAD_GLOBAL
STORE_GLOBAL

LOAD_FIELD
STORE_FIELD

LOAD_CONST

ADD
SUB
MUL
DIV
MOD
NEG

CMP_EQ
CMP_NE
CMP_LT
CMP_LE
CMP_GT
CMP_GE

AND
OR
XOR
NOT

SHL
SHR

JMP
JMP_IF
JMP_IF_NOT

CALL
CALL_VIRTUAL
CALL_INTERFACE

RETURN

NEW
NEW_ARRAY

GET_ELEMENT
SET_ELEMENT

CAST
TYPE_CHECK

THROW

NATIVE_CALL
```

---

# 28. NOP

```text
NOP
```

Не выполняет действий.

Используется для:

- выравнивания;
- патчинга;
- отладки;
- будущих оптимизаций.

---

# 29. PUSH

Помещает значение на operand stack.

Пример:

```text
PUSH constantIndex
```

---

# 30. POP

Удаляет верхнее значение:

```text
POP
```

---

# 31. DUP

Дублирует верхнее значение:

```text
DUP
```

Например:

```text
[10]
```

после `DUP`:

```text
[10, 10]
```

---

# 32. Local Operations

```text
LOAD_LOCAL index
STORE_LOCAL index
```

`LOAD_LOCAL`:

```text
local[index] → stack
```

`STORE_LOCAL`:

```text
stack → local[index]
```

---

# 33. Global Operations

```text
LOAD_GLOBAL index
STORE_GLOBAL index
```

Используются для доступа к глобальным переменным.

---

# 34. Field Operations

```text
LOAD_FIELD fieldIndex
STORE_FIELD fieldIndex
```

Для объекта:

```AZONE
player.health
```

VM выполняет:

```text
object → LOAD_FIELD health
```

---

# 35. Arithmetic

Поддерживаются:

```text
ADD
SUB
MUL
DIV
MOD
NEG
```

Для числовых типов VM использует type metadata и правила языка.

---

# 36. Comparison

```text
CMP_EQ
CMP_NE
CMP_LT
CMP_LE
CMP_GT
CMP_GE
```

Результат:

```text
bool
```

---

# 37. Bitwise Operations

```text
AND
OR
XOR
NOT
SHL
SHR
```

Применимы к целочисленным типам.

---

# 38. Conditional Branch

```text
JMP offset
```

Безусловный переход.

```text
JMP_IF offset
```

Переход, если верхнее значение является `true`.

```text
JMP_IF_NOT offset
```

Переход, если значение `false`.

---

# 39. Function Call

```text
CALL functionIndex argumentCount
```

Перед вызовом аргументы помещаются на operand stack.

Пример:

```text
PUSH 10
PUSH 20
CALL add 2
```

После выполнения результат остаётся на stack.

---

# 40. Virtual Call

Для виртуального метода:

```text
CALL_VIRTUAL methodIndex argumentCount
```

VM получает объект и определяет фактическую реализацию через type metadata / vtable.

---

# 41. Interface Call

Для интерфейсного метода:

```text
CALL_INTERFACE interfaceMethodIndex argumentCount
```

VM выполняет interface dispatch.

---

# 42. Return

```text
RETURN
```

или:

```text
RETURN value
```

В конкретном encoding `RETURN` может использовать metadata функции для определения наличия возвращаемого значения.

---

# 43. Object Allocation

```text
NEW typeIndex
```

Создаёт объект указанного типа.

VM должна:

1. определить размер объекта;
2. определить alignment;
3. выделить память;
4. инициализировать object header;
5. установить Type ID;
6. выполнить конструктор, если это предусмотрено инструкциями.

---

# 44. Array Allocation

```text
NEW_ARRAY typeIndex
```

Размер массива берётся со stack:

```text
[length]
NEW_ARRAY int32
```

Создаётся массив:

```text
int32[length]
```

---

# 45. Array Access

```text
GET_ELEMENT
SET_ELEMENT
```

При обычном безопасном режиме VM выполняет bounds check.

При выходе за границы:

```text
ArrayIndexOutOfBounds
```

В `unsafe` / native-контексте допускаются специальные операции без стандартной проверки.

---

# 46. Cast

```text
CAST typeIndex
```

Выполняет преобразование типа согласно правилам [[Specification 0.3]].

Недопустимое преобразование вызывает runtime error или исключение.

---

# 47. Type Check

```text
TYPE_CHECK typeIndex
```

Проверяет совместимость объекта с указанным типом.

Возвращает:

```text
bool
```

---

# 48. Exceptions

`.azb` должен поддерживать исключения.

Минимальная модель:

```text
THROW
```

Для каждой функции может существовать таблица handlers.

Handler содержит:

```text
tryStart
tryEnd
handlerStart
handlerEnd
exceptionType
```

Пример:

```text
try
  ↓
instructions
  ↓
throw
  ↓
handler
```

---

# 49. Exception Unwinding

При `THROW` VM:

1. получает исключение;
2. ищет handler текущей функции;
3. если handler найден — передаёт управление ему;
4. если нет — unwinding текущего frame;
5. переходит к caller;
6. продолжает поиск;
7. при отсутствии handler в entry point завершает программу с необработанным исключением.

---

# 50. GC Metadata

Поскольку AZONE LANG поддерживает managed memory, VM должна знать, где находятся ссылки.

GC Metadata содержит информацию о safepoints.

Например:

```text
Function #4
Bytecode Offset: 120

Reference Stack Slots:
    0
    3

Reference Locals:
    2
    5
```

Это позволяет использовать precise GC.

---

# 51. GC Safepoints

Safepoint может находиться:

- перед вызовом функции;
- после allocation;
- в циклах;
- перед native call;
- в других точках, определённых VM.

VM может остановить поток на safepoint для работы GC.

---

# 52. Relocation Table

Relocation Table используется для ссылок, которые невозможно окончательно определить во время компиляции.

Например:

```text
Native Function
External Symbol
Module Symbol
Runtime Symbol
```

Каждая relocation entry содержит:

```text
offset
type
symbol
addend
```

---

# 53. Symbol Table

Symbol Table содержит экспортируемые и необходимые внешние символы.

Например:

```text
main
System.Console.write
Memory.allocate
Kernel.schedule
```

Symbol Table особенно важна для:

- модулей;
- библиотек;
- native API;
- FFI;
- будущего AZONE OS.

---

# 54. Module Metadata

`.azb` должен хранить:

```text
Module Name
Module Version
Language Version
Required Modules
Exported Symbols
Imported Symbols
```

Пример:

```text
module System.Memory

version 0.1

requires:
    System.Core

exports:
    allocate
    free
```

---

# 55. Global Data

Global Data содержит:

- статические переменные;
- глобальные константы;
- runtime initialization data.

Пример:

```AZONE
static int32 counter = 0;
```

может быть представлено в Global Data.

---

# 56. Static Initialization

Если модуль имеет статическую инициализацию, `.azb` должен содержать ссылку на соответствующую функцию.

VM выполняет её до использования модуля в соответствии с правилами module initialization.

---

# 57. Class Table

Class Table содержит:

```text
Class ID
Type ID
Base Class
Interface List
Field Range
Method Range
Flags
```

Например:

```text
Class: Player
Base: Entity
Interfaces:
    Drawable
    Serializable
```

---

# 58. Field Table

Каждое поле содержит:

```text
Field Name
Type ID
Flags
Offset
Owner Type
```

Для managed object offset может быть runtime-dependent.

Для native/ABI-compatible типов offset может быть фиксированным.

---

# 59. Method Table

Метод содержит:

```text
Method Name
Owner Type
Function ID
Flags
Virtual Slot
Interface Slot
```

Это позволяет VM выполнять:

```text
CALL
CALL_VIRTUAL
CALL_INTERFACE
```

---

# 60. Debug Information

Debug Section является необязательной.

Может содержать:

```text
Source File
Line Number
Column
Bytecode Offset
Function
Local Variable
Type
Scope
```

Пример:

```text
Bytecode 120 → Main.az:42:5
```

Debug Information используется:

- debugger;
- stack trace;
- profiler;
- runtime diagnostics;
- IDE.

---

# 61. Source Mapping

VM должна иметь возможность построить:

```text
Bytecode Offset
        ↓
Function
        ↓
Source File
        ↓
Line
        ↓
Column
```

Это особенно важно для исключений.

---

# 62. Stack Trace

При исключении VM должна иметь возможность сформировать:

```text
Exception: NullReference

at Player.update()
at Game.tick()
at Game.run()
at main()
```

Если `.azb` содержит Debug Information, stack trace должен включать исходные файлы и строки.

---

# 63. Checksum

`.azb` может содержать checksum.

Минимальная реализация должна позволять определить повреждение файла.

Checksum вычисляется по определённым секциям файла, исключая само поле checksum.

Алгоритм checksum будет окончательно закреплён в отдельной технической документации реализации.

---

# 64. Bytecode Validation

Перед исполнением VM обязана проверить:

- Magic Number;
- версию;
- размеры файла;
- секции;
- offsets;
- индексы;
- Constant Pool;
- Type IDs;
- Function IDs;
- Field IDs;
- Method IDs;
- bytecode boundaries;
- branch targets;
- stack requirements;
- local indices;
- exception ranges;
- GC metadata;
- relocations;
- checksum, если присутствует.

---

# 65. Проверка Stack Safety

VM должна предотвращать:

```text
stack underflow
stack overflow
invalid stack state
```

Компилятор должен генерировать корректный bytecode.

VM дополнительно валидирует его перед исполнением или в режиме runtime validation.

---

# 66. Invalid Bytecode

При обнаружении некорректного байткода VM не должна пытаться его исполнять.

Пример:

```text
AZB Validation Error

Invalid CALL:
Function ID 999999 does not exist.
```

---

# 67. Bounds Checking

Все обращения по индексам `.azb` должны проверяться:

```text
constantIndex < constantCount
typeIndex < typeCount
functionIndex < functionCount
fieldIndex < fieldCount
methodIndex < methodCount
```

---

# 68. Безопасность `.azb`

`.azb` не должен автоматически предоставлять произвольный доступ к памяти хоста.

Portable bytecode работает через VM abstraction.

Прямой доступ к:

```text
pointer
address
native memory
MMIO
CPU-specific instructions
```

доступен только через соответствующие unsafe/native механизмы.

---

# 69. Portable vs Native Bytecode

AZONE LANG разделяет:

```text
Portable VM Bytecode
```

и:

```text
Native Code
```

Portable bytecode может выполняться на любой архитектуре, для которой существует совместимая AZONE VM.

Native code зависит от:

```text
CPU
ABI
Operating System
Runtime
```

Поэтому `.azb` с native-секциями должен содержать target information.

---

# 70. Native Call

Инструкция:

```text
NATIVE_CALL
```

используется для обращения к native function.

Схема:

```text
AZB
 ↓
VM
 ↓
Native Bridge
 ↓
ABI Adapter
 ↓
Native Function
```

Это соответствует модели [[Specification 0.5]].

---

# 71. Native ABI

Для первой системной реализации:

```text
x86-64
Linux
System V AMD64 ABI
```

В будущем:

```text
AArch64 ABI
RISC-V64 ABI
```

---

# 72. Entry Point

`.azb` может содержать entry point.

Обычно:

```az
main()
```

представляется как Function ID.

Header содержит:

```text
entryPoint
```

VM запускает его при запуске executable `.azb`.

---

# 73. Libraries

`.azb` может быть библиотекой.

В этом случае:

```text
IsLibrary = true
```

и entry point может отсутствовать.

Библиотека предоставляет:

```text
exports
```

и может иметь:

```text
imports
```

---

# 74. Executable

Исполняемый `.azb`:

```text
IsExecutable = true
```

может содержать:

```text
entryPoint
```

и запускается через:

```text
azvm program.azb
```

---

# 75. Module Loading

VM loader выполняет:

```text
Open .azb
 ↓
Read Header
 ↓
Validate Header
 ↓
Read Section Table
 ↓
Validate Sections
 ↓
Load Metadata
 ↓
Resolve Dependencies
 ↓
Resolve Symbols
 ↓
Apply Relocations
 ↓
Validate Bytecode
 ↓
Initialize Runtime
 ↓
Execute
```

---

# 76. Module Dependencies

Модуль может требовать другие модули:

```text
Application
 ├── System.Core
 ├── System.Memory
 └── Graphics
      └── System.Core
```

Loader должен построить dependency graph.

Циклические зависимости в первой версии не рекомендуются.

---

# 77. Lazy Loading

Lazy module loading может быть добавлен в будущем.

В таком режиме модуль загружается только при первом обращении.

Это особенно важно для будущей AZONE OS.

---

# 78. Memory Mapping

Формат должен быть спроектирован с возможностью будущего memory-mapped loading.

Некоторые read-only секции потенциально могут быть отображены непосредственно из файла:

```text
String Table
Constant Pool
Read-only metadata
Bytecode
Debug Information
```

Однако VM не обязана использовать mmap в первой реализации.

---

# 79. Compression

Сжатие секций не является обязательным для версии 0.6.

В будущем может быть добавлено:

```text
Compressed Bytecode
Compressed Debug Info
Compressed Strings
```

с соответствующим section flag.

---

# 80. Signing

Формат резервирует возможность цифровой подписи:

```text
IsSigned
```

Криптографический механизм не входит в MVP.

В будущем возможно:

```text
Signature
Public Key ID
Algorithm ID
```

Это может быть особенно полезно для безопасной загрузки системных компонентов AZONE OS.

---

# 81. Version Compatibility

VM должна соблюдать следующие правила:

```text
Format Major incompatible
    → reject

Unsupported required feature
    → reject

Unknown optional section
    → may ignore

Compatible minor extension
    → may load
```

---

# 82. Unknown Sections

VM должна уметь пропускать неизвестные необязательные секции:

```text
offset
size
```

Это позволяет расширять формат без немедленной поломки старых инструментов.

Если неизвестная секция помечена как обязательная:

```text
required = true
```

старая VM должна отклонить файл.

---

# 83. AZB Validation Levels

В будущем можно поддержать несколько уровней:

```text
LEVEL 0 — Header validation
LEVEL 1 — Structural validation
LEVEL 2 — Bytecode validation
LEVEL 3 — Type validation
LEVEL 4 — Security validation
```

Для запуска executable должен быть выполнен как минимум полный structural + bytecode validation.

---

# 84. Compiler Requirements

Компилятор AZONE LANG должен:

1. генерировать валидный `.azb`;
2. создавать корректные таблицы;
3. разрешать ссылки;
4. рассчитывать stack requirements;
5. создавать GC metadata;
6. создавать exception metadata;
7. создавать debug information при необходимости;
8. создавать relocation information для unresolved native symbols;
9. устанавливать корректные версии;
10. устанавливать target information.

---

# 85. VM Requirements

VM должна поддерживать:

- `.azb` loader;
- validator;
- Constant Pool;
- Type Table;
- Function Table;
- Class Table;
- Field Table;
- Method Table;
- module loader;
- operand stack;
- local slots;
- call stack;
- heap;
- object allocation;
- arrays;
- virtual dispatch;
- interface dispatch;
- exceptions;
- GC safepoints;
- native calls;
- entry point;
- debug stack traces.

---

# 86. MVP Instruction Set

Для первого рабочего прототипа достаточно:

```text
NOP

PUSH
POP
DUP

LOAD_LOCAL
STORE_LOCAL

LOAD_CONST

ADD
SUB
MUL
DIV
MOD

CMP_EQ
CMP_NE
CMP_LT
CMP_LE
CMP_GT
CMP_GE

JMP
JMP_IF
JMP_IF_NOT

CALL
RETURN

NEW
LOAD_FIELD
STORE_FIELD

GET_ELEMENT
SET_ELEMENT

CAST
TYPE_CHECK

THROW
```

После этого добавляются:

```text
CALL_VIRTUAL
CALL_INTERFACE
NATIVE_CALL
GC Metadata
Relocations
```

---

# 87. Пример байткода

Исходный код:

```AZONE
function main()
{
    int32 a = 10;
    int32 b = 20;

    return a + b;
}
```

Концептуально:

```text
PUSH 10
STORE_LOCAL 0

PUSH 20
STORE_LOCAL 1

LOAD_LOCAL 0
LOAD_LOCAL 1

ADD

RETURN
```

---

# 88. Пример вызова метода

AZONE LANG:

```AZONE
player.move(10, 20);
```

Концептуально:

```text
LOAD_LOCAL 0       ; player
PUSH 10
PUSH 20
CALL_VIRTUAL move 2
POP
```

---

# 89. Пример создания объекта

Исходный код:

```AZONE
Player player = new Player();
```

Байткод:

```text
NEW Player
STORE_LOCAL 0
```

Конструктор может вызываться отдельной инструкцией:

```text
CALL constructor
```

или быть встроен в семантику `NEW` в зависимости от окончательной реализации VM.

Для первой версии рекомендуется разделить allocation и constructor call на логические операции, чтобы VM сохраняла ясную модель:

```text
NEW
CALL constructor
```

---

# 90. Bytecode Verification

Для каждой функции VM может построить control-flow graph:

```text
Entry
 ↓
Instruction
 ↓
Branch ──→ Instruction
 ↓
Instruction
 ↓
Return
```

Verifier проверяет, что все пути имеют корректное состояние stack.

Например:

```text
Path A → stack depth 2
Path B → stack depth 1
          ↓
       invalid merge
```

Такой bytecode должен быть отклонён.

---

# 91. Deterministic Compilation

При одинаковых:

```text
Source
Compiler Version
Compiler Extensions
Compiler Options
Target
```

компилятор должен по возможности создавать одинаковый `.azb`.

Это облегчает:

- тестирование;
- reproducible builds;
- диагностику;
- сравнение бинарников.

---

# 92. `.azb` и AZONE OS

`.azb` является одним из ключевых форматов будущей AZONE OS.

Планируется, что:

```text
AZONE OS
 │
 ├── Kernel
 ├── Drivers
 ├── Process Manager
 ├── Memory Manager
 ├── File System
 ├── Network Stack
 ├── Display Server
 ├── GUI
 └── Applications
```

смогут использовать AZONE LANG и его runtime.

При этом системные компоненты могут использовать:

```text
Managed AZONE
Native AZONE
Unsafe AZONE
```

в зависимости от требований.

---

# 93. Kernel Bytecode

В будущем возможно существование специального режима:

```text
Freestanding AZB
```

Он не требует полноценного hosted runtime.

Например:

```text
Kernel
 ↓
AZONE VM / Native backend
 ↓
Hardware
```

Конкретная модель исполнения kernel-кода будет определена в Specification 0.9 и последующих системных спецификациях.

---

# 94. VM Bytecode ≠ CPU Machine Code

Важно:

```text
.azb
```

не является машинным кодом x86-64.

Например:

```text
ADD
```

в `.azb` означает операцию виртуальной машины.

VM самостоятельно преобразует её в необходимые операции CPU.

В будущем возможна JIT/AOT-компиляция:

```text
AZB
 ↓
JIT
 ↓
x86-64 machine code
```

или:

```text
AZB
 ↓
AOT Compiler
 ↓
Native executable
```

---

# 95. Возможность JIT

JIT не входит в MVP.

Архитектура `.azb` должна позволять:

```text
Interpreter
JIT
AOT
```

использовать один и тот же байткод.

---

# 96. Возможность оптимизации

В будущем VM может оптимизировать:

```text
constant folding
dead code elimination
inline calls
virtual call devirtualization
stack optimization
type specialization
JIT compilation
```

Однако `.azb` должен оставаться валидным независимо от наличия оптимизатора.

---

# 97. Reserved Opcodes

Диапазон opcode должен содержать резерв.

Например:

```text
0x00–0x7F = standard
0x80–0xEF = reserved
0xF0–0xFF = extension/runtime
```

Точная схема будет закреплена после формирования полной таблицы opcode.

---

# 98. Extension Opcodes

`.azm` может в будущем добавлять новые bytecode instructions.

Однако такие инструкции должны иметь:

```text
Opcode ID
Version
Operand Format
Stack Effect
Runtime Requirement
```

VM должна знать, какие расширения поддерживаются.

---

# 99. Stack Effect Metadata

Каждая инструкция должна иметь определённый stack effect.

Например:

```text
PUSH       : +1
POP        : -1
DUP        : +1
ADD        : -1
LOAD_LOCAL : +1
STORE_LOCAL: -1
```

Это используется verifier.

---

# 100. Stack Effect Examples

```text
PUSH 10
```

до:

```text
[]
```

после:

```text
[10]
```

`ADD`:

```text
[a, b]
```

становится:

```text
[a + b]
```

то есть:

```text
-2 + 1 = -1
```

---

# 101. Type Safety

Bytecode verifier должен проверять совместимость типов там, где это возможно статически.

Например:

```text
ADD
```

должен применяться к совместимым числовым значениям.

Однако часть проверок может выполняться runtime для динамических или extension types.

---

# 102. Error Categories

VM должна различать:

```text
InvalidAZB
UnsupportedAZBVersion
InvalidOpcode
InvalidConstant
InvalidType
InvalidFunction
InvalidMethod
InvalidField
StackUnderflow
StackOverflow
InvalidBranch
InvalidReference
InvalidExceptionTable
InvalidGCMetadata
NativeSymbolNotFound
ModuleNotFound
```

---

# 103. ABI Boundary

`.azb` может взаимодействовать с native code:

```text
AZB Value
 ↓
VM Representation
 ↓
ABI Adapter
 ↓
Native Representation
```

Преобразование должно учитывать [[Specification 0.5]].

---

# 104. Future Generic Types

Generic metadata в `.azb` пока не фиксируется окончательно.

В будущем Type Table может быть расширена:

```text
Generic Type
Generic Parameter
Generic Constraint
Instantiation
```

---

# 105. Future Reflection

Type Table и Method Table создают основу для reflection.

В будущем программа сможет получать:

```text
Type
Fields
Methods
Interfaces
Attributes
```

runtime-информацию.

Reflection API будет определён позднее.

---

# 106. Форматирование и инструменты

Планируются инструменты:

```text
azc
azvm
az
azdump
azdis
azverify
```

### `azdump`

Показывает структуру `.azb`.

### `azdis`

Дизассемблирует bytecode.

### `azverify`

Проверяет `.azb` без запуска.

Пример:

```text
azverify program.azb
```

Результат:

```text
AZB VALID
Format: 0.6
Functions: 12
Types: 8
Bytecode: 4.2 KB
```

---

# 107. Дизассемблер

Пример:

```text
0000  PUSH_CONST 0
0005  STORE_LOCAL 0
0007  PUSH_CONST 1
000C  STORE_LOCAL 1
000E  LOAD_LOCAL 0
0010  LOAD_LOCAL 1
0012  ADD
0013  RETURN
```

Это не является частью исполнения, но является важнейшим инструментом разработки VM.

---

# 108. Требования к расширяемости

Формат `.azb` должен позволять добавлять:

- новые типы;
- новые инструкции;
- новые metadata;
- новые ABI;
- новые native backends;
- новые debug formats;
- новые GC mechanisms.

При этом расширение не должно ломать старые `.azb`, если VM может безопасно игнорировать необязательную функциональность.

---

# 109. Минимальная архитектура AZB Loader

```text
AZBLoader
 ├── HeaderReader
 ├── SectionReader
 ├── ConstantPoolLoader
 ├── StringTableLoader
 ├── TypeLoader
 ├── FunctionLoader
 ├── ClassLoader
 ├── ModuleLoader
 ├── RelocationLoader
 ├── DebugLoader
 └── Validator
```

После загрузки:

```text
AZBLoader
    ↓
Module
    ↓
VM
```

---

# 110. Итоговая архитектура

На уровне проекта:

```text
┌─────────────────────┐
│       .az source    │
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│      Compiler       │
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│       .azb          │
│                     │
│ Header              │
│ Sections            │
│ Constants           │
│ Types               │
│ Classes             │
│ Functions           │
│ Bytecode            │
│ Exceptions          │
│ GC Metadata         │
│ Symbols             │
│ Debug Info          │
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│     AZONE VM        │
│                     │
│ Loader              │
│ Validator           │
│ Interpreter         │
│ Stack               │
│ Call Stack           │
│ Heap                │
│ GC                  │
│ Runtime             │
│ Native Bridge       │
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ Host / AZONE OS     │
└─────────────────────┘
```

---

# 111. Что фиксирует Specification 0.6

Данная спецификация фиксирует:

- `.azb` как основной бинарный формат AZONE LANG;
- Header;
- версии;
- target information;
- Section Table;
- Constant Pool;
- String Table;
- Type Table;
- Function Table;
- Class Table;
- Field Table;
- Method Table;
- Module Metadata;
- Global Data;
- Bytecode;
- Exception Data;
- GC Metadata;
- Relocation Table;
- Symbol Table;
- Debug Information;
- checksum;
- stack-based instruction model;
- opcode/operand model;
- bytecode validation;
- module loading;
- native bridge;
- portable/native distinction;
- основу для interpreter/JIT/AOT;
- совместимость с будущей AZONE OS.

---

# 112. Что пока НЕ фиксируется окончательно

Оставляются для следующих спецификаций:

- конкретная внутренняя реализация VM;
- точный Call Stack;
- точная модель VM frames;
- GC algorithm;
- threading model;
- VM scheduler;
- native ABI implementation;
- `.azm` extension mechanism;
- полноценная Native API;
- kernel integration;
- process model;
- OS-specific system calls;
- JIT;
- AOT compiler;
- точная бинарная таблица всех opcode;
- generics metadata;
- reflection API.

---

# 113. Следующая спецификация

Следующая спецификация:
## [[Specification 0.7 |Specification 0.7 — AZONE VM Execution Model]]

Она должна определить непосредственно устройство виртуальной машины:

```text
VM
├── Instruction Pointer
├── Operand Stack
├── Call Stack
├── Stack Frames
├── Local Variables
├── Heap
├── Global Storage
├── Constant Pool
├── Type Metadata
├── Object Model
├── Exception System
├── GC Integration
├── Native Bridge
├── Threads
└── Runtime State
```

Также в [[Specification 0.7 |0.7]] необходимо определить:

- жизненный цикл VM;
- создание и уничтожение VM;
- выполнение `.azb`;
- stack frames;
- вызовы функций;
- виртуальные вызовы;
- создание объектов;
- работу heap;
- runtime exceptions;
- взаимодействие VM и GC;
- VM threads;
- thread stacks;
- synchronization;
- safepoints;
- native calls;
- VM shutdown;
- режимы **Hosted / Freestanding**;
- API для embedding VM в C++-приложения.

Это станет фундаментом уже непосредственно **работающей AZONE VM**, а не только формата байткода.