# AZONE LANG Specification 0.4

## Object Model, Classes & Modules

**Проект:** AZONE LANG  
**Версия спецификации:** 0.4  
**Статус:** Draft  
**Целевая платформа:** собственная операционная система на базе Linux kernel  
**Тип языка:** статически типизированный, объектно-ориентированный  
**VM:** стековая  
**Реализация:** C++20

---
# 1. Назначение

Specification 0.4 определяет объектную модель AZONE LANG и систему организации исходного кода.

Документ определяет:

- классы;
- объекты;
- поля;
- методы;
- конструкторы;
- деструкторы;
- наследование;
- полиморфизм;
- виртуальные методы;
- интерфейсы;
- модификаторы доступа;
- статические члены;
- свойства;
- `this`;
- `base`;
- пространства имён;
- модули;
- `import`;
- `export`;
- видимость;
- разрешение методов;
- идентичность объектов.

Низкоуровневое физическое расположение объектов в памяти определяется в [[Specification 0.5]].

---
# 2. Основная объектная модель

AZONE LANG использует объектную модель, основанную на классах.

Концептуально:

```
Class 
  │ 
  ├── Fields 
  ├── Methods 
  ├── Constructors 
  ├── Properties 
  └── Static Members 
       │ 
       ▼ 
    Object
```

Класс описывает структуру и поведение объектов.

Объект является экземпляром класса.

---
# 3. Объявление класса

Базовый синтаксис:

```AZONE
class Player { 
	int32 health; 
	string name; 
}
```

Класс может содержать:

```
fields
methods
constructors
destructors
properties
nested types
static members
```

---
# 4. Создание объекта

Для создания объекта используется:

```
new
```

Пример:

```AZONE
Player player = new Player();
```

При создании объекта:

1. выделяется необходимая память;
2. создаётся объект;
3. инициализируется объектная метаинформация;
4. выполняется конструктор;
5. возвращается ссылка на объект.

Точная схема выделения памяти определяется Memory Model.

---
# 5. Объектная идентичность

Каждый созданный объект имеет собственную идентичность.

Например:

```AZONE
Player a = new Player(); 
Player b = new Player();
```

Даже если:

```AZONE
a.health == b.health
```

объекты `a` и `b` являются разными объектами.

Проверка идентичности должна отличаться от проверки равенства значений.

Предусматривается концепция:

```
reference equality
value equality
```

Точный оператор и API будут определены отдельно.

---
# 6. Поля

Поле хранит состояние объекта.

```AZONE
class Player { 
	int32 health; 
	string name; 
}
```

Каждый экземпляр `Player` имеет собственные:

```
health
name
```

если поле не объявлено как `static`.

---
# 7. Доступ к полям

Для доступа к полю используется оператор:

`.`

Пример:

```AZONE
player.health = 100; 
player.name = "Zenith";
```

Если поле недоступно из текущего контекста, компилятор должен выдать ошибку доступа.

---
# 8. Методы

Метод является функцией, принадлежащей классу.

```AZONE
class Player { 
	int32 health; 
	
	void heal(int32 amount) { 
		health = health + amount; 
	} 
}
```

Вызов:

```AZONE
player.heal(20);
```

Метод имеет доступ к состоянию текущего объекта через `this`.

---
# 9. this

Внутри нестатического метода доступен специальный идентификатор:

```
this
```

Он представляет текущий объект.

Пример:

```AZONE
class Player { 
	int32 health; 
	
	void damage(int32 amount) { 
		this.health = this.health - amount; 
	} 
}
```

В большинстве случаев `this` может быть опущен:

```AZONE
health = health - amount;
```

---
# 10. this как ссылка

`this` концептуально является ссылкой на текущий объект:

```
reference<Class>
```

Однако его внутреннее представление определяется VM и ABI.

`this` нельзя переназначить:

```AZONE
this = other;
```

Такой код должен быть ошибкой компиляции.

---
# 11. Конструкторы

Конструктор используется для инициализации объекта.

Предлагаемый синтаксис:

```AZONE
class Player { 
	int32 health; 
	
	Player(int32 initialHealth) { 
		health = initialHealth; 
	} 
}
```

Создание:

```AZONE
Player player = new Player(100);
```

Конструктор должен иметь имя класса.

---
# 12. Конструктор по умолчанию

Если класс не содержит пользовательских конструкторов, компилятор может автоматически предоставить конструктор без параметров:

```
Player()
```

Например:

```AZONE
class Player { 
	int32 health; 
}

Player player = new Player();
```

Точные правила автоматической генерации конструктора должны быть определены компилятором.

---
# 13. Перегрузка конструкторов

Класс может иметь несколько конструкторов.

```AZONE
class Player { 
	Player() { 
		... 
	} 
	
	Player(int32 health) { 
		... 
	} 
	
	Player(int32 health, string name) { 
		... 
	} 
}
```

Выбор конструктора выполняется по типам аргументов.

---
# 14. Инициализация полей

Поля могут иметь значения по умолчанию:

```AZONE
class Player {
    int32 health = 100;
    bool active = true;
}
```

Порядок инициализации должен быть определён как:

1. allocation
2. default field initialization
3. base-class initialization
4. constructor initialization
5. constructor body

Точный порядок при наследовании будет закреплён в ABI/Memory Model.

---
# 15. Деструктор

Предусматривается деструктор:

`~ClassName()`

Пример:

```AZONE
class File { 
	~File() { 
		close(); 
	} 
}
```

Деструктор предназначен для освобождения ресурсов, связанных с объектом.

Важно:

**деструктор не обязан напрямую означать освобождение памяти.**

Освобождение памяти и уничтожение объекта являются отдельными операциями runtime.

Это позволит использовать Garbage Collector.

---
# 16. Финализатор

В случае управляемой памяти может существовать дополнительный механизм:

```
finalizer
```

Финализатор отличается от деструктора.

```
Destructor
    │
    └── детерминированное завершение объекта

Finalizer
    │
    └── runtime-managed cleanup
```

Конкретный синтаксис финализатора пока не фиксируется.

---
# 17. Наследование

AZONE LANG поддерживает наследование классов.

Предварительный синтаксис:

```AZONE
class Dragon : Animal { 
	... 
}
```

В этом примере:

```
Animal 
   ▲ 
   │ 
Dragon
```

`Dragon` наследует доступные элементы `Animal`.

---
# 18. Одиночное наследование классов

Для основной объектной модели выбирается **одиночное наследование классов**.

То есть:

```AZONE
class Dragon : Animal { 
}
```

разрешено.

Но:

```AZONE
class Dragon : Animal, Creature { 
}
```

если `Animal` и `Creature` являются классами, запрещается.

Причина:

- более простой object layout;
- более простой VM;
- предсказуемый ABI;
- отсутствие сложного diamond inheritance;
- упрощение виртуального dispatch.

Множественное поведение обеспечивается интерфейсами.

---
# 19. Наследование интерфейсов

Класс может реализовывать несколько интерфейсов:

```AZONE
class Dragon : Animal, IFlyable, IWarrior { 
}
```

Здесь:

```
Animal       = base class
IFlyable     = interface
IWarrior     = interface
```

Количество интерфейсов не ограничивается моделью одиночного наследования классов.

---
# 20. base

В производном классе доступен:
```
base
```

Он позволяет обращаться к базовой части объекта.

Например:

```AZONE
class Dragon : Animal { 
	void speak() { 
		base.speak(); 
	} 
}
```

`base` также используется для вызова базового конструктора.

---
# 21. Переопределение методов

Метод базового класса может быть объявлен виртуальным:

```AZONE
class Animal { 
	virtual void speak() { 
		... 
	} 
}
```

Производный класс может переопределить его:

```AZONE
class Dragon : Animal { 
	override void speak() { 
		... 
	} 
}
```

Использование `override` является обязательным для явного переопределения виртуального метода.

Это позволяет компилятору обнаруживать ошибки:

```AZONE
override void speek() { 
}
```

если `speek` не существует в базовом классе.

---
# 22. Virtual

`virtual` означает, что вызов метода может выполняться динамически.

```AZONE
class Animal { 
	virtual void speak() { 
		... 
	} 
}
```

При наличии:

```AZONE
Animal animal = new Dragon(); 
animal.speak();
```

будет вызван метод `Dragon.speak()`.

---
# 23. Non-virtual методы

Метод без `virtual` по умолчанию является обычным методом.

```AZONE
class Animal { 
	void eat() { 
		... 
	} 
}
```

Вызов может быть разрешён статически.

Это позволяет компилятору выполнять оптимизацию:

```
direct call
```

вместо:

```
virtual dispatch
```

---
# 24. Запрет переопределения

Для методов может быть предусмотрен модификатор:

```
final
```

Пример:

```AZONE
class Animal { 
	virtual final void identity() { 
		... 
	} 
}
```

После этого производный класс не может переопределить метод.

`final` также может использоваться для классов:

```AZONE
final class SystemKernel { 
}
```

Такой класс нельзя наследовать.

---
# 25. Абстрактные классы

Предусматриваются абстрактные классы:

```AZONE
abstract class Animal { 
	abstract void speak(); 
}
```

Абстрактный класс нельзя создать напрямую:

```AZONE
new Animal();
```

это ошибка.

Производный класс должен реализовать все обязательные абстрактные методы.

---
# 26. Интерфейсы

Интерфейс описывает контракт поведения.

Пример:

```AZONE
interface IFlyable { 
	void fly(); 
}
```

Класс:

```AZONE
class Dragon : IFlyable { 
	void fly() { 
		... 
	} 
}
```

Интерфейс не является классом и не имеет собственного состояния экземпляра.

---
# 27. Интерфейсные поля

Обычные нестатические поля в интерфейсах запрещены.

Например:

```AZONE
interface IFlyable { 
	int32 speed; 
}
```

должно быть ошибкой.

Интерфейс определяет поведение, а не состояние объекта.

Константы и статические элементы интерфейса могут быть разрешены отдельными правилами.

---
# 28. Интерфейсные методы

Методы интерфейса по умолчанию являются абстрактными:

```AZONE
interface IPrintable { 
	void print(); 
}
```

Реализация:

```AZONE
class Document : IPrintable { 
	void print() { 
		... 
	} 
}
```

---
# 29. Проверка реализации интерфейса

Компилятор должен проверять соответствие:

```
method name
parameter count
parameter types
return type
visibility
```

Если класс объявлен:

```AZONE
class Document : IPrintable {
}
```

но не реализует:

```
print()
```

компиляция должна завершиться ошибкой.

---
# 30. Полиморфизм

AZONE LANG поддерживает полиморфизм через:

```
class inheritance
virtual methods
interfaces
```

Пример:

```AZONE
Animal animal = new Dragon(); 

animal.speak();
```

Статический тип:

```
Animal
```

Фактический тип:

```
Dragon
```

Вызов виртуального метода определяется фактическим типом объекта.

---
# 31. Виртуальный dispatch

Для виртуальных методов VM должна поддерживать механизм динамического вызова.

Концептуально:

```AZONE
Object 
  │ 
  ▼ 
Type Information 
  │
  ▼ 
Virtual Method Table 
  │ 
  ▼ 
Target Method
```

Физический формат vtable не фиксируется Specification 0.4.

Он будет определён в:

Specification 0.5 — Memory Model & ABI

---
# 32. Модификаторы доступа

AZONE LANG должен поддерживать:

```
public
private
protected
internal
```

---
# 33. public

`public` означает доступность из любого разрешённого модуля.

```AZONE
class Player { 
	public int32 health; 
}
```

---
# 34. private

`private` ограничивает доступ текущим типом.

```AZONE
class Player { 
	private int32 health; 
}
```

Производный класс не получает прямой доступ к `private` полю.

---
# 35. protected

`protected` позволяет доступ:

- внутри класса;
- внутри производных классов.

```AZONE
class Animal {
    protected int32 age;
}

class Dragon : Animal {
    void grow() {
        age++;
    }
}
```

---
# 36. internal

`internal` ограничивает доступ текущим модулем.

```AZONE
internal class KernelMemory { 
}
```

Это особенно важно для системных компонентов AZONE OS.

Например:

```
kernel.memory
kernel.scheduler
kernel.ipc
```

могут скрывать внутренние реализации друг от друга.

---
# 37. Значения доступа по умолчанию

Для класса:

```
class
```

члены по умолчанию:

```
private
```

Для интерфейса методы по умолчанию:

```
public
```

Это уменьшает вероятность случайного раскрытия внутреннего состояния.

---
# 38. Статические поля

Статическое поле принадлежит классу, а не отдельному объекту.

```AZONE
class Player { 
	static int32 count; 
}
```

Доступ:

```
Player.count;
```

Все экземпляры класса используют одно значение:

```
Player.count
```

---
# 39. Статические методы

```AZONE
class Math { 
	static int32 add(int32 a, int32 b) { 
		return a + b; 
	} 
}
```

Вызов:

```AZONE
Math.add(10, 20);
```

Статический метод не имеет `this`.

Попытка использовать `this` внутри static-метода является ошибкой.

---
# 40. Статический конструктор

Для инициализации статического состояния предусматривается:

```
static constructor
```

Пример:

```AZONE
class System { 
	static int32 version; static constructor() { 
		version = 1; 
	} 
}
```

Статический конструктор выполняется runtime один раз перед первым использованием соответствующего типа.

Точная модель инициализации будет определена в модуле Runtime.

---
# 41. Свойства

AZONE LANG должен поддерживать свойства как высокоуровневый механизм доступа к данным.

Пример:

```AZONE
class Player { 
	private int32 health; public int32 Health { 
		get { 
			return health; 
		} 
		
		set { 
			health = value; 
		} 
	} 
}
```

Использование:

```AZONE
player.Health = 100;
int32 hp = player.Health;
```

Свойство не обязано иметь собственное поле.

Оно может быть полностью вычисляемым.

---
# 42. Read-only свойства

Допускается свойство только для чтения:

```AZONE
public int32 Health { 
	get { 
		return health; 
	} 
}
```

Попытка:

```AZONE
player.Health = 100;
```

должна быть ошибкой.

---
# 43. Автоматические свойства

Для простых случаев предусматривается сокращённая форма:

```AZONE
public string Name { get; set; }
```

Компилятор автоматически создаёт скрытое поле.

Такое поле не должно быть напрямую доступно пользователю.

---
# 44. Методы с одинаковым именем

Методы могут перегружаться.

```AZONE
class Logger { 
	void log(string message) { 
		... 
	} 
	
	void log(int32 value) { 
		... 
	} 
}
```

Выбор выполняется по сигнатуре вызова.

---
# 45. Сигнатура метода

Сигнатура должна учитывать:

```
method name
parameter count
parameter types
generic parameters
```

Возвращаемый тип не должен использоваться единственным критерием перегрузки.

Например:

```AZONE
int32 get(); 
string get();
```

не может существовать как две перегрузки только из-за разных возвращаемых типов.

---
# 46. Модули

AZONE LANG вводит систему модулей.

Модуль является логической единицей компиляции и распространения кода.

Пример:

```
azone.graphics
azone.network
azone.kernel
azone.fs
```

Модуль может содержать:

```
classes
interfaces
functions
enums
constants
other modules
```

---
# 47. Объявление модуля

Предварительный синтаксис:

```AZONE
module azone.graphics;
```

После объявления все элементы файла принадлежат этому модулю.

---
# 48. Import

Для использования другого модуля:

```AZONE
import azone.graphics;
```

Например:

```AZONE
module game; 

import azone.graphics; 

class Game { 
}
```

---
# 49. Импорт конкретного символа

Может поддерживаться:

```AZONE
import azone.graphics.Canvas;
```

Это позволяет импортировать только необходимый символ.

---
# 50. Алиасы

Для конфликтующих имён допускаются алиасы:

```AZONE
import graphics as gfx;
```

Использование:

```AZONE
gfx.Canvas canvas;
```

Точная форма алиасов будет уточнена в Module System.

---
# 51. Export

Модуль может экспортировать API:

```AZONE
export class Window {
}
```

Символ без `export` не считается частью публичного API модуля, если иное не установлено правилами видимости.

---
# 52. Public API модуля

Пример:

```AZONE
module azone.graphics;

export class Window {
}

class WindowImpl {
}
```

Внешний код может использовать:

```
Window
```

но не:

```
WindowImpl
```

Это позволяет разделить:

```
Public API
Internal Implementation
```

---
# 53. Namespace

Namespace является пространством имён, а module — единицей компиляции.

Это разные понятия.

Например:

```
Module:
azone.graphics
```

может содержать namespace:

```
azone.graphics.render
```

Namespace предназначен для организации имён.

Module предназначен для организации и компиляции исходного кода.

---
# 54. Полное имя символа

Символ должен иметь квалифицированное имя.

Например:

```AZONE
azone.graphics.Window
```

или:

```AZONE
azone.kernel.memory.Page
```

Это предотвращает конфликты между большими проектами.

---
# 55. Циклические импорты

Модули не должны создавать неконтролируемые циклические зависимости.

Например:

```
A → B
B → A
```

должно либо:

1. поддерживаться специальной системой разрешения зависимостей;

либо:

2. приводить к диагностируемой ошибке.

Для первой версии рекомендуется запретить циклические зависимости между модулями.

---
# 56. Module Dependency Graph

Компилятор должен строить граф зависимостей:

```
        Application
        /    |     \
       ▼     ▼      ▼
    Graphics Network  FS
       │       │      │
       └───────┴──────┘
               │
             Core
```

Компиляция выполняется с учётом этого графа.

---
# 57. Компиляционная единица

Файл `.az` является исходной единицей, но не обязательно конечной единицей распространения.

Несколько файлов могут принадлежать одному модулю:

```
graphics/
    window.az
    canvas.az
    surface.az
```

Все они могут принадлежать:

```AZONE
azone.graphics
```

---
# 58. Module Metadata

Скомпилированный модуль должен содержать метаданные:

```
Module Name
Module Version
Compiler Version
Language Version
Target
Dependencies
Exported Symbols
Type Information
ABI Information
```

Эти данные могут храниться в `.azb` или отдельном module metadata-файле.

---
# 59. Module Versioning

Модули должны иметь версии.

Например:

```
azone.graphics 1.2.0
```

В будущем система зависимостей должна позволять указывать совместимые диапазоны версий.

Например концептуально:

```
azone.graphics >= 1.2
```

Формат dependency manifest будет определён позднее.

---
# 60. Entry Point

Исполняемый модуль может иметь точку входа:

```AZONE
int main() {
    ...
}
```

Для системных компонентов могут существовать другие entry points.

Например:

```
process entry
service entry
driver entry
kernel service entry
```

Поэтому `main()` не должен быть единственным допустимым типом точки входа на уровне всей платформы.

---
# 61. Пример объектной модели

```AZONE
module game.characters;

interface IDamageable {
    void damage(int32 amount);
}

class Character {
    protected int32 health;

    Character(int32 health) {
        this.health = health;
    }

    virtual void speak() {
        print("Character");
    }

    public int32 Health {
        get {
            return health;
        }
    }
}

class Dragon : Character, IDamageable {

    Dragon(int32 health)
        : base(health) {
    }

    override void speak() {
        print("Grr-r!");
    }

    public void damage(int32 amount) {
        health = health - amount;
    }
}
```

Использование:

```AZONE
import game.characters.Dragon;

Dragon dragon = new Dragon(100);

dragon.speak();
dragon.damage(20);

print(dragon.Health);
```

---
# 62. Требования к VM

VM должна поддерживать как минимум:

```
object allocation
field access
method call
static method call
constructor call
virtual dispatch
interface dispatch
type checking
reference handling
```

Необходимые инструкции байткода будут определены в [[Specification 0.6]].

---
# 63. Требования к компилятору

Компилятор должен выполнять:

```
class declaration validation
inheritance validation
interface validation
method overload resolution
override validation
access control
constructor resolution
property validation
module dependency resolution
import resolution
export validation
```

---
# 64. Запреты

Компилятор должен запрещать:

- наследование от `final` класса;
- переопределение `final` метода;
- создание экземпляра `abstract` класса;
- доступ к `private` члену извне;
- доступ к `protected` члену из несвязанного контекста;
- использование неизвестного метода;
- вызов static-метода через `this`;
- циклические module dependencies, если они не поддерживаются;
- реализацию интерфейса с несовместимой сигнатурой;
- перегрузку только по возвращаемому типу.

---
# 65. Будущая совместимость с системным программированием

Объектная модель не должна препятствовать созданию низкоуровневых структур.

В дальнейшем должны быть предусмотрены типы:

```
struct
union
packed struct
extern struct
```

Они будут необходимы для:

- Linux ABI;
- системных вызовов;
- сетевых пакетов;
- аппаратных регистров;
- файловых форматов;
- MMIO;
- взаимодействия с C ABI.

Эти возможности будут описаны отдельно.

---
# 66. Разделение языка и платформы

Объектная модель является частью:

```
AZONE LANG
```

а следующие сущности относятся к платформе:

```
AZONE OS Process
AZONE OS Service
AZONE OS Window
AZONE OS IPC
AZONE OS Device
AZONE OS Driver
```

Язык должен предоставлять механизмы, на которых они могут быть построены, но не включать их непосредственно в ядро языка.

---
# 67. Архитектура объектной системы

Предварительная схема:

```
                 Class Metadata
                       │
                       ▼
                  ┌─────────┐
                  │  Class  │
                  └────┬────┘
                       │
             ┌─────────┼─────────┐
             ▼         ▼         ▼
          Fields    Methods   Interfaces
             │         │         │
             └─────────┼─────────┘
                       ▼
                    Object
                       │
             ┌─────────┼─────────┐
             ▼         ▼         ▼
          State     Type Info   Methods
                       │
                       ▼
                    Runtime
```

Физический layout будет определён в [[Specification 0.5]].

---
# 68. Итоговые требования Specification 0.4

В AZONE LANG должны поддерживаться:

```
Classes                    REQUIRED
Objects                    REQUIRED
Fields                     REQUIRED
Methods                    REQUIRED
Constructors               REQUIRED
Destructors                REQUIRED
Inheritance                REQUIRED
Single class inheritance   REQUIRED
Multiple interfaces        REQUIRED
Virtual methods            REQUIRED
Override                   REQUIRED
Abstract classes           REQUIRED
Interfaces                 REQUIRED
Access modifiers           REQUIRED
Static members             REQUIRED
Properties                 REQUIRED
this                       REQUIRED
base                       REQUIRED
Modules                    REQUIRED
Namespaces                 REQUIRED
Import                     REQUIRED
Export                     REQUIRED
Module visibility          REQUIRED
Method overloading         REQUIRED
Virtual dispatch           REQUIRED
Interface dispatch         REQUIRED
```

---
# 69. Переносимые вопросы

Следующие вопросы не фиксируются окончательно в Specification 0.4:

```
Object memory layout
Field alignment
Object header
VTable layout
Interface table layout
GC metadata
Reference representation
Calling convention
Exception ABI
Atomic object operations
Thread safety
Struct layout
Union layout
Native ABI
Bytecode instructions
```

Они будут определены в последующих спецификациях.

---
# 70. Следующая спецификация

Следующий этап:

# [[Specification 0.5 |AZONE LANG Specification 0.5]]

## Memory Model & ABI

В ней необходимо определить фундаментальную модель памяти AZONE LANG:

```
Stack
Heap
Object Header
Object Layout
Field Layout
Alignment
Padding
References
Pointers
Arrays
Strings
Structs
Unions
Memory Ownership
Garbage Collection
Object Lifetime
Destructors
Finalizers
Thread Local Storage
Atomics
Volatile
Memory Barriers
Calling Convention
ABI
C ABI
Native ABI
FFI
```

Именно **[[Specification 0.5]]** станет связующим звеном между объектной моделью языка и реальной памятью/аппаратурой. После неё уже можно будет достаточно точно проектировать формат `.azb` и инструкции стековой VM.