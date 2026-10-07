# AZONE LANG Specification 0.5

## Memory Model & ABI

**Проект:** AZONE LANG  
**Версия спецификации:** 0.5  
**Статус:** Draft  
**Целевая платформа:** AZONE OS / Linux kernel  
**Язык реализации компилятора и VM:** C++20  
**Модель исполнения:** Stack-based VM  
**Типизация:** Static  
**Исходный файл:** `.az`  
**Compiled bytecode:** `.azb`

---

# 1. Назначение

Specification 0.5 определяет модель памяти AZONE LANG и правила взаимодействия языка с аппаратной платформой.

Документ определяет:

- Stack;
- Heap;
- глобальную память;
- объектную память;
- layout объектов;
- alignment;
- padding;
- ссылки;
- указатели;
- массивы;
- строки;
- структуры;
- unions;
- время жизни объектов;
- владение памятью;
- Garbage Collection;
- ручное управление памятью;
- native memory;
- ABI;
- calling convention;
- C ABI;
- FFI;
- атомарные операции;
- memory ordering;
- `volatile`;
- взаимодействие с низкоуровневым кодом.

---

# 2. Основной принцип

AZONE LANG должен поддерживать два режима работы:

```text
Managed
Native
```

### Managed

Runtime управляет временем жизни объектов.

Используется для:

- обычных приложений;
- GUI;
- сервисов;
- высокоуровневых компонентов ОС.

### Native

Программист получает прямой контроль над памятью.

Используется для:

- ядра;
- драйверов;
- MMIO;
- системных структур;
- boot-компонентов;
- allocator;
- runtime;
- взаимодействия с C/C++ ABI.

---

# 3. Адресное пространство

AZONE LANG различает:

```text
virtual address
physical address
native address
```

Язык не должен автоматически считать виртуальный адрес физическим адресом.

Для доступа к физической памяти должны существовать специальные системные API.

---

# 4. Основные области памяти

Исполняемая программа концептуально содержит:

```text
┌──────────────────────────┐
│        Code / Text       │
├──────────────────────────┤
│      Read-only Data      │
├──────────────────────────┤
│        Global Data       │
├──────────────────────────┤
│           Heap           │
│            ↓             │
│                          │
│            ↑             │
│          Stack           │
└──────────────────────────┘
```

Фактическая организация зависит от платформы и runtime.

---

# 5. Stack

Stack используется для:

- локальных переменных;
- параметров функций;
- временных значений;
- VM stack frames;
- return information;
- служебного состояния вызова.

Пример:

```azone
void test() {
    int32 value = 100;
}
```

`value` может размещаться на stack.

Точная стратегия размещения определяется компилятором.

---

# 6. Stack Frame

Каждый вызов функции концептуально создаёт frame:

```text
┌──────────────────────┐
│ Local Variables      │
├──────────────────────┤
│ Temporary Values     │
├──────────────────────┤
│ Saved State          │
├──────────────────────┤
│ Return Information   │
└──────────────────────┘
```

Для VM frame может быть виртуальным.

Для native backend frame может соответствовать реальному machine stack.

---

# 7. Heap

Heap предназначен для динамически создаваемых объектов.

Например:

```azone
Player player = new Player();
```

Объект `Player` обычно размещается в managed heap.

Heap управляется runtime.

---

# 8. Global Memory

Глобальные данные включают:

```text
static fields
global constants
module state
runtime metadata
native globals
```

Они существуют дольше отдельных stack frames.

---

# 9. Object Memory

Объект состоит концептуально из:

```text
┌──────────────────────┐
│ Object Header        │
├──────────────────────┤
│ Base Class Fields    │
├──────────────────────┤
│ Derived Fields       │
├──────────────────────┤
│ Alignment / Padding  │
└──────────────────────┘
```

Конкретный layout будет зависеть от ABI.

---

# 10. Object Header

Managed object может иметь служебный header.

Предварительно он может содержать:

```text
Type Information
GC Metadata
Object Flags
Synchronization Data
```

Например:

```text
┌───────────────┐
│ Type Pointer  │
├───────────────┤
│ GC Metadata   │
├───────────────┤
│ Object Flags  │
├───────────────┤
│ Object Data   │
└───────────────┘
```

Фактический размер header определяется ABI/runtime.

---

# 11. Type Information

Каждый managed object должен быть связан с информацией о своём типе.

Type metadata может содержать:

```text
Type ID
Type Name
Base Type
Fields
Methods
Interfaces
Size
Alignment
GC Information
```

Это необходимо для:

- type checking;
- virtual dispatch;
- interface dispatch;
- reflection;
- GC.

---

# 12. Alignment

Каждый тип имеет alignment requirement.

Например концептуально:

```text
uint8   → alignment 1
uint16  → alignment 2
uint32  → alignment 4
uint64  → alignment 8
```

Фактические требования зависят от target ABI.

---

# 13. Padding

Компилятор может вставлять padding между полями.

Например:

```azone
class Example {
    uint8 a;
    uint64 b;
}
```

Физический layout может отличаться от порядка только по размеру полей:

```text
a
padding
b
```

Padding не считается отдельным пользовательским полем.

---

# 14. Field Offset

Каждое поле имеет offset относительно начала соответствующей части объекта.

Концептуально:

```text
object + field_offset
```

Для managed объектов пользователь не должен полагаться на конкретные offsets без специального ABI/interop-модификатора.

---

# 15. Layout Guarantee

Обычный `class` не обязан иметь ABI-stable layout.

Если требуется строго определённое расположение:

```text
struct
packed struct
extern struct
```

должны использоваться специальные конструкции.

---

# 16. Value Types

Value types хранят собственное значение.

Примеры:

```text
int32
uint64
float64
bool
char
struct
```

При присваивании value type обычно копируется значение.

```azone
int32 a = 10;
int32 b = a;
```

Изменение `b` не должно изменять `a`.

---

# 17. Reference Types

Reference types представляют объекты.

Например:

```azone
Player a = new Player();
Player b = a;
```

В этом случае:

```text
a ─────┐
       ▼
   ┌────────┐
   │ Player │
   └────────┘
       ▲
       │
b ─────┘
```

`a` и `b` указывают на один объект.

---

# 18. Reference Representation

В языке ссылка может быть представлена как:

```text
managed reference
```

Она отличается от:

```text
pointer<T>
```

Managed reference контролируется runtime.

Pointer является низкоуровневым адресом.

---

# 19. Pointer

Для системного программирования:

```azone
pointer<int32>
```

представляет указатель на `int32`.

Операции:

```azone
int32 value = 10;

pointer<int32> p = &value;

int32 x = *p;
```

Разыменование указателя разрешается только в соответствующем безопасном/unsafe контексте.

---

# 20. Unsafe Context

Низкоуровневые операции требуют:

```azone
unsafe {
    ...
}
```

Например:

```azone
unsafe {
    int32 value = 10;
    pointer<int32> p = &value;

    *p = 20;
}
```

Компилятор должен явно обозначать операции, которые могут нарушить memory safety.

---

# 21. Address

Тип:

```text
address
```

представляет машинный адрес.

Он не обязан содержать информацию о типе данных.

Например:

```azone
address addr;
```

Использование address требует особой осторожности.

---

# 22. Pointer Conversion

Преобразования между:

```text
pointer<T>
address
```

могут быть разрешены только явно.

Например концептуально:

```azone
address addr = pointerToAddress(p);
pointer<uint8> p = addressToPointer<uint8>(addr);
```

Неявное преобразование запрещается.

---

# 23. Null

Указатель или ссылка может иметь отсутствие значения:

```text
null
```

Например:

```azone
Player player = null;
```

Попытка обращения к `null` должна приводить к определённой runtime error.

Для native pointers поведение зависит от unsafe-контекста.

---

# 24. Arrays

AZONE LANG поддерживает несколько видов массивов.

### Static array

```azone
int32[10] values;
```

Размер известен во время компиляции.

### Dynamic array

```azone
int32[] values;
```

Размер определяется во время выполнения.

---

# 25. Static Array Layout

Static array должен содержать элементы последовательно:

```text
element[0]
element[1]
element[2]
...
```

Для типа `T`:

```text
address(element[n]) =
base + n * sizeof(T)
```

при отсутствии специальных packing rules.

---

# 26. Dynamic Array

Dynamic array концептуально содержит:

```text
data pointer
length
capacity
```

Например:

```text
┌──────────────┐
│ data         │
│ length       │
│ capacity     │
└──────────────┘
```

Реальный layout является частью runtime ABI.

---

# 27. Bounds Checking

В managed режиме доступ:

```azone
values[index]
```

должен проверять границы.

При:

```text
index < 0
```

или:

```text
index >= length
```

возникает runtime error.

---

# 28. Unsafe Array Access

В `unsafe` режиме может существовать unchecked-доступ.

Например концептуально:

```azone
unsafe {
    int32 value = uncheckedIndex(values, index);
}
```

Такой механизм необходим для высокопроизводительных системных операций.

---

# 29. Strings

`string` является managed reference type.

Строка должна хранить:

```text
characters/data
length
encoding metadata, если требуется
```

Базовая кодировка языка:

```text
UTF-8
```

---

# 30. String Immutability

По умолчанию `string` является immutable.

Например:

```azone
string a = "Hello";
```

Операция изменения символа напрямую запрещена.

Для изменяемых буферов используются специальные типы:

```text
StringBuilder
char buffer
byte buffer
```

---

# 31. String Memory

Строковые данные могут содержать:

```text
length
data
```

Наличие terminating `\0` не является обязательным для обычного `string`.

Для C ABI должен существовать отдельный механизм:

```text
cstring
```

или эквивалентный native representation.

---

# 32. Struct

`struct` является value type.

Пример:

```azone
struct Point {
    int32 x;
    int32 y;
}
```

Использование:

```azone
Point p;
p.x = 10;
p.y = 20;
```

---

# 33. Struct Copy

При обычном присваивании:

```azone
Point a;
Point b = a;
```

копируется значение структуры.

Если структура содержит reference types, копируется сама ссылка, а не объект.

---

# 34. Packed Struct

Для бинарных форматов и аппаратного взаимодействия предусматривается:

```text
packed struct
```

Например:

```azone
packed struct Header {
    uint8 type;
    uint16 length;
}
```

Padding между полями минимизируется согласно packing rules.

Использование packed structures может иметь performance penalties.

---

# 35. Extern Struct

Для совместимости с внешним ABI:

```azone
extern struct LinuxHeader {
    ...
}
```

Layout определяется внешним ABI.

Это позволит описывать:

- Linux structures;
- C structures;
- device structures;
- syscall structures.

---

# 36. Union

Для совместного использования одной области памяти предусматривается:

```text
union
```

Пример:

```azone
union Value {
    int32 integer;
    float32 decimal;
}
```

Все поля union используют одну область памяти.

---

# 37. Union Safety

Union является низкоуровневым механизмом.

Безопасное использование должно учитывать активный член union.

Для произвольной интерпретации памяти требуется `unsafe`.

---

# 38. Ownership

AZONE LANG должен различать:

```text
ownership
borrowing
sharing
```

Однако первая версия языка не обязана реализовывать полноценную borrow checker систему.

Минимальная модель:

```text
Managed references
Native pointers
Explicit ownership APIs
```

---

# 39. Garbage Collector

Managed runtime может использовать Garbage Collector.

GC отвечает за объекты, которые:

```text
не имеют достижимых managed references
```

Пример:

```azone
void test() {
    Player player = new Player();
}
```

После выхода из функции объект может стать недостижимым.

Когда именно он будет освобождён, определяет GC.

---

# 40. GC не гарантирует немедленное освобождение

Следует различать:

```text
object becomes unreachable
```

и:

```text
memory is reclaimed
```

Они не обязательно происходят одновременно.

---

# 41. GC Root

GC должен отслеживать корни:

```text
stack references
global references
static references
runtime handles
native handles
thread-local references
```

Из них определяется достижимость объектов.

---

# 42. Native Memory

Native memory не обязана управляться GC.

Например:

```azone
unsafe {
    pointer<uint8> buffer = nativeAlloc(4096);
    ...
    nativeFree(buffer);
}
```

Ответственность за native memory лежит на программисте или соответствующем allocator.

---

# 43. Memory Allocator

Runtime должен предоставлять абстракцию allocator.

Минимальный интерфейс:

```text
allocate(size, alignment)
reallocate(pointer, size)
free(pointer)
```

Конкретная реализация может быть:

```text
GC Heap
System Heap
Kernel Heap
Arena
Pool
Stack Allocator
Custom Allocator
```

---

# 44. Custom Allocator

AZONE LANG должен позволять библиотекам и системным компонентам предоставлять собственные allocators.

Например:

```azone
Allocator kernelAllocator;
```

Это особенно важно для AZONE OS.

Kernel может использовать:

```text
slab allocator
page allocator
object pools
```

не изменяя язык.

---

# 45. Lifetime

В языке существуют различные модели lifetime:

```text
Stack lifetime
Managed lifetime
Explicit native lifetime
Static lifetime
Thread lifetime
```

Компилятор должен различать их там, где это необходимо для безопасности.

---

# 46. Destructor Lifetime

Деструктор managed объекта не должен использоваться как механизм гарантированного времени освобождения памяти.

Если необходим детерминированный cleanup, рекомендуется использовать:

```text
close()
dispose()
release()
```

или специальный RAII/ownership-механизм, если он будет добавлен в будущем.

---

# 47. Thread Local Storage

Язык должен предусматривать:

```text
thread-local
```

данные.

Например:

```azone
threadlocal int32 errno;
```

Каждый thread получает отдельную копию.

---

# 48. Atomic Types

Для многопоточного программирования предусматриваются атомарные типы:

```text
atomic<int32>
atomic<int64>
atomic<usize>
```

Операции должны поддерживать как минимум:

```text
load
store
exchange
compare_exchange
fetch_add
fetch_sub
```

---

# 49. Memory Ordering

Атомарные операции должны поддерживать memory ordering:

```text
relaxed
acquire
release
acq_rel
seq_cst
```

Названия и точная семантика должны быть совместимы с архитектурой memory model.

---

# 50. Volatile

Для аппаратных регистров и MMIO предусматривается:

```text
volatile
```

Например:

```azone
volatile pointer<uint32> register;
```

`volatile` означает, что compiler не может свободно удалить или объединить соответствующие memory accesses.

`volatile` **не является механизмом синхронизации между потоками**.

---

# 51. Memory Barriers

Native API должен предоставлять memory barriers:

```text
fence()
loadFence()
storeFence()
fullFence()
```

Точная реализация зависит от target architecture.

---

# 52. ABI

ABI определяет взаимодействие бинарных компонентов.

AZONE ABI должен описывать:

```text
type representation
calling convention
parameter passing
return values
stack layout
register usage
alignment
object layout
name mangling
exception handling
thread-local storage
FFI
```

---

# 53. Calling Convention

Для native backend первоначальной основной архитектурой является:

```text
x86-64
```

На первом этапе должна быть предусмотрена совместимость с системным ABI Linux x86-64.

В дальнейшем:

```text
AArch64
RISC-V 64
```

---

# 54. VM Calling Convention

Внутри VM вызовы могут использовать отдельную convention.

Концептуально:

```text
push arguments
call function
function creates frame
return value pushed
```

VM ABI не обязан совпадать с native ABI.

---

# 55. Native Calling Convention

При вызове native функции runtime должен знать:

```text
argument locations
return location
register preservation
stack alignment
calling convention
```

Для x86-64 Linux первоначально целевой ABI должен соответствовать System V AMD64 ABI.

---

# 56. C ABI

AZONE LANG должен иметь возможность вызывать функции C ABI.

Например концептуально:

```azone
extern "C" int32 strlen(pointer<uint8> value);
```

Это необходимо для:

- Linux;
- POSIX API;
- системных библиотек;
- драйверов;
- существующих native libraries.

---

# 57. Native Functions

Native function может быть объявлена:

```azone
extern native void kernelWrite(uint8 value);
```

Реализация предоставляется runtime или native library.

---

# 58. FFI

Foreign Function Interface должен позволять:

```text
AZONE → C
AZONE → native ABI
native ABI → AZONE
```

Обратный вызов:

```text
C → AZONE
```

тоже должен быть возможен для совместимых функций.

---

# 59. ABI-Stable Types

Для FFI рекомендуются:

```text
int8
int16
int32
int64
uint8
uint16
uint32
uint64
float32
float64
pointer<T>
address
extern struct
```

Managed classes напрямую ABI-stable не считаются.

---

# 60. Class ABI

Обычный:

```azone
class Player {
}
```

не должен считаться стабильной binary ABI structure.

Причина:

компилятор может изменить:

```text
object header
vtable
field layout
GC metadata
```

без изменения исходного API.

---

# 61. Native ABI Class

Если объект должен пересекать ABI boundary, должна существовать специальная native representation.

Например:

```text
extern struct
```

или специально объявленный ABI-stable тип.

---

# 62. Name Mangling

Перегруженные функции требуют уникальных бинарных имён.

Например:

```azone
log(int32)
log(string)
```

могут преобразовываться во внутренние символы:

```text
_Z...log...
```

Формат mangling должен быть детерминированным.

Для:

```text
extern "C"
```

name mangling должен отключаться.

---

# 63. Endianness

AZONE LANG должен определять target endianness.

Первоначальная основная архитектура:

```text
x86-64 little-endian
```

Portable code не должен предполагать конкретный byte order без явного указания.

---

# 64. Integer Representation

Целочисленные типы должны использовать двоичное представление.

Размеры:

```text
int8    = 8 bits
int16   = 16 bits
int32   = 32 bits
int64   = 64 bits

uint8   = 8 bits
uint16  = 16 bits
uint32  = 32 bits
uint64  = 64 bits
```

---

# 65. Pointer Size

Размер:

```text
pointer<T>
address
usize
isize
```

зависит от target architecture.

На x86-64:

```text
pointer size = 64 bits
```

---

# 66. Type Size API

Runtime/compiler должны предоставлять:

```text
sizeof(T)
alignof(T)
```

Например:

```azone
usize size = sizeof(int32);
usize alignment = alignof(int32);
```

Для managed classes `sizeof` должен возвращать только разрешённое ABI/runtime значение либо быть запрещён для обычного объекта.

---

# 67. Memory Safety Levels

AZONE LANG концептуально имеет уровни:

```text
SAFE
UNSAFE
NATIVE
```

### SAFE

Запрещены опасные операции.

### UNSAFE

Разрешаются:

```text
pointers
address manipulation
raw memory
```

### NATIVE

Разрешается непосредственная работа с ABI и внешним кодом.

---

# 68. Safe/Unsafe Boundary

Переход из:

```text
safe → unsafe
```

должен быть явным.

Например:

```azone
unsafe {
    pointer<uint8> ptr = ...;
}
```

Это позволяет компилятору и аудиторам кода быстро находить потенциально опасные участки.

---

# 69. Memory Model для AZONE OS

Модель должна позволять создавать:

```text
Kernel
Driver
Process
Thread
Scheduler
Memory Manager
File System
Network Stack
Display Server
GUI
User Applications
```

при этом высокоуровневые компоненты могут использовать managed memory, а низкоуровневые компоненты — native memory.

---

# 70. Freestanding Runtime

AZONE LANG должен поддерживать режим:

```text
freestanding
```

В котором нет требования наличия полной стандартной библиотеки или OS runtime.

Это необходимо для:

```text
bootloader
kernel
early kernel
runtime initialization
drivers
embedded systems
```

---

# 71. Hosted Runtime

В hosted режиме доступны:

```text
GC
threads
files
network
processes
GUI
standard library
```

Конкретный набор зависит от платформы.

---

# 72. Runtime Layers

Предлагается следующая архитектура:

```text
┌──────────────────────────┐
│      AZONE Application   │
├──────────────────────────┤
│     AZONE Standard Lib   │
├──────────────────────────┤
│      AZONE Runtime       │
├──────────────────────────┤
│       Native ABI         │
├──────────────────────────┤
│      Operating System    │
├──────────────────────────┤
│          Hardware        │
└──────────────────────────┘
```

Для AZONE OS некоторые слои будут реализовываться непосредственно средствами AZONE LANG.

---

# 73. Memory Model и VM

Stack-based VM не обязана напрямую использовать native machine stack.

VM может иметь:

```text
Operand Stack
Call Stack
Heap
Global Storage
Constant Pool
Type Metadata
```

Например:

```text
┌─────────────────────┐
│ Global Storage      │
├─────────────────────┤
│ Heap                │
├─────────────────────┤
│ Call Stack           │
├─────────────────────┤
│ Operand Stack        │
└─────────────────────┘
```

---

# 74. Native Calls из VM

При вызове native function:

```text
VM
 │
 ▼
Native Bridge
 │
 ▼
ABI Adapter
 │
 ▼
Native Function
```

Bridge должен преобразовывать VM representation в native ABI representation.

---

# 75. GC и Native Calls

GC должен учитывать native code, если native code временно удерживает managed references.

Для этого потребуется механизм:

```text
GC Handle
```

Например концептуально:

```text
strong handle
weak handle
pinned handle
```

Точные типы будут определены в Runtime Specification.

---

# 76. Pinning

Некоторые native операции требуют стабильного адреса объекта.

Для этого предусматривается:

```text
pin
unpin
```

Pinned object не должен перемещаться GC во время pinning.

Это особенно важно для:

```text
DMA
MMIO
native I/O
C API
```

---

# 77. Memory Ownership между AZONE и Native

При передаче буфера:

```text
AZONE → Native
```

должно быть однозначно определено:

```text
who owns memory?
who frees memory?
how long is memory valid?
can native code retain pointer?
```

ABI не должен оставлять эти вопросы неявными.

---

# 78. Общий принцип ABI

Любая ABI boundary должна иметь явно определённые:

```text
Type
Ownership
Lifetime
Alignment
Calling Convention
Error Convention
Threading Rules
```

---

# 79. Ошибки памяти

Runtime должен уметь диагностировать как минимум:

```text
null reference
invalid pointer
out-of-bounds access
use-after-free
double free
invalid alignment
invalid native call
```

В safe режиме многие из этих ситуаций должны быть предотвращены до выполнения.

---

# 80. Требования Specification 0.5

AZONE LANG должен поддерживать:

```text
Stack                       REQUIRED
Heap                        REQUIRED
Global memory               REQUIRED
Object metadata             REQUIRED
Alignment                   REQUIRED
Padding                     REQUIRED
References                  REQUIRED
Pointers                    REQUIRED
Address type                REQUIRED
Static arrays               REQUIRED
Dynamic arrays              REQUIRED
Structs                     REQUIRED
Packed structs              REQUIRED
Unions                      REQUIRED
Managed memory              REQUIRED
Native memory               REQUIRED
GC                          REQUIRED
Unsafe blocks               REQUIRED
Atomics                     REQUIRED
Volatile                    REQUIRED
Memory barriers             REQUIRED
Native ABI                  REQUIRED
C ABI                       REQUIRED
FFI                         REQUIRED
Freestanding mode           REQUIRED
Hosted mode                 REQUIRED
```

---

# 81. Что остаётся определить

Следующие вопросы переносятся в следующие спецификации:

```text
GC algorithm
Exception ABI
Exact object header
Exact vtable layout
Exact interface table
Exact bytecode representation
Native opcode set
VM instruction set
Binary module format
Debugger format
Reflection metadata
Thread scheduler
Async model
```

---

# 82. Следующая спецификация

Следующий этап:

# AZONE LANG Specification 0.6

## Bytecode & `.azb` Binary Format

В Specification 0.6 будет определён **реальный байткод AZONE LANG**.

Будут описаны:

```text
.azb Header
Magic Number
Version
Target Architecture
Flags
Constant Pool
String Pool
Type Table
Function Table
Class Table
Method Table
Field Table
Module Metadata
Global Data
Bytecode Section
Debug Section
Exception Section
GC Metadata
Relocation Table
Symbol Table
Checksum
```

А также непосредственно инструкции стековой VM:

```text
PUSH
POP
DUP
LOAD
STORE
LOAD_LOCAL
STORE_LOCAL
LOAD_FIELD
STORE_FIELD
ADD
SUB
MUL
DIV
MOD
CMP
JMP
JMP_IF
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

После **0.6** у нас уже появится формальное бинарное представление программы, а затем в **0.7** можно будет спроектировать саму VM-интерпретаторную машину и её execution model.