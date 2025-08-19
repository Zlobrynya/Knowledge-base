Система для управление памятью в Swift, строится на основе подсчета ссылок которые ссылаются на объект

Преимущество:
- Работает на этапе компиляции, компилятор самостоятельно добавляет команды `retain` и `release` -> нету накладных расходов в runtime
Недостатки:
- Начинает течь при **Retain Cycle**
#### **Типы ссылок**

Существует 3 вида ссылок:
- **strong** - сильная ссылка, в случае если счетчик == 0, происходит deinit объекта, пока счетчик >= 1, объект держится в памяти
- **weak** - **слабая** ссылка (не увеличивает счётчик ссылок). Она автоматически становится `nil` при освобождении объекта.
- **unowned** - **небезопасная** слабая ссылка. Не обнуляется при освобождении объекта. Обращение к ней после освобождения вызовет краш (в отличие от безопасного `weak`).

У каждого reference объекта в заголовке памяти находится поля для strong и unowned счетчиков.

```
  HeapObject {
    isa
    InlineRefCounts {
      atomic<InlineRefCountBits> {
        strong RC + unowned RC + flags
        OR
        HeapObjectSideTableEntry*
      }
    }
  }

  HeapObjectSideTableEntry {
    SideTableRefCounts {
      object pointer
      atomic<SideTableRefCountBits> {
        strong RC + unowned RC + weak RC + flags
      }
    }
  }
```
В случае когда на этот объект начинают ссылаться слабо, создается **SideTable**, в которую переносятся все счетчики.
https://github.com/swiftlang/swift/blob/main/stdlib/public/SwiftShims/swift/shims/RefCount.h#L78 

**SideTable** создается при:
- Возникновении weak ссылки
- Переполнении счетчиков для strong и unowned

#### **Retain Cycles**

Это циклы сильных ссылок, классический пример:
````swift
class First {
	var second: Second
}

class Second {
	var first: First
}
````
При таком коде, у First будет strong == 2 и у Second strong == 2 -> оба объекта будут висеть в памяти, даже после выхода со scope где они используются. 
Для исправления этого цикла следует использовать **weak**:
````swift
class First {
	weak var second: Second?
}

class Second {
	var first: First
}
````

Теперь после выхода из scope, где используется оба этих класса, оба объекта будут успешно освобождены.