# Лекция 1.2. Lock-free stack и Michael-Scott queue

## План лекции

1. Неблокирующиеся алгоритмы
2. Lock-free Treiber Stack
3. Michael-Scott Queue

## Mutual exclusion
Aka `mutex` или `lock`, только один поток может держать блокировку.

### Atomic counter
```kotlin
val l = Lock()
var counter = 0

fun incAndGet(): Int {
    l.lock()
    try {
    return ++counter
    } finally {
        l.unlock()
    }
}
```
Эта блокировка грубая. Нет гарантии прогресса в том случае, если CPU отберут, потому что все другие потоки будут ждать наш поток.

## Lock-freedom
Гарантирует прогресс в системе. Базовая операция (реализована в железе): `CAS (Compare-And-Set)` – `CAS(p, old, new)` атомарно проверяет, что значение по адресу `p` атомарно совпадает с `old` и заменяет его на `new`.

### Counter
```kotlin
fun getAndInc(): Int {
    while (true) {
        cur := c
        if CAS(&c, cur, cur + 1) {
            return c
        }
    }
}
```

### Stack
```kotlin
fun pop(): Int {
    head := H
    while (true) {
        if CAS(&H, head, head?.Next) {
                if head == null: return null
                return head.Value
            }
        }
    }
}

fun push(x: Int) {
    while (true) {  
        head := H
        newHead := Node {Value: x, Next: head}
        if CAS(&H, head, newHead): return
    }
}

fun top(): Int? {
    head := H
    return head?.Value
}
```
__Последовательная согласованность (Sequential consistency)__ – это такое последовательное исполнение, которое учитывает порядок внутри потоков, но не учитывает синхронизацию между потоками.

### Elimination for Stack
Оптимизация следующая: заведём дополнительный массив фиксированного размера, `push` сначала пойдёт в его рандомную ячейку (или рядом, если та была уже занята), а `pop` смотрит несколько рядом стоящих ячеек и ищет занятую. Если операция дождалась (`push` – `pop`'а, `pop` – нового элемента), то всё хорошо, иначе идём в стек и действуем по выше описанному алгоритму.

Таким образом, `push` и `pop` могут встретиться, поменяться данными и не дергать стек.

### Queue
Операция добавления элемента должна переписать `Next` у `Tail` и подвинуть `Tail` на последний элемент. Если мы перепишем `Next`, но не успеем подвинуть `Tail` (какой-то другой поток тоже начал писать) всё сломается.
__Helping__: пусть если поток хочет положить элемент в конец очереди, а у последнего элемента уже есть `Next`, он просто поможет подвинуть `Tail` на элемент вперёд и опять попытается добавить свой элемент уже в следующей итерации. Тогда при добавлении элемента и передвижении `Tail` его `CAS` для передвижения `Tail` не пройдёт – хвост уже перенесли.

```kotlin
fun enqueue (x: int) {
    newTail := Node { Value = x, N = null}
    while (true) { // CAS loop
        tail := T
        if CAS(&tail.N, null, newTail) {
            // 'newTail' just added, move the tail forward
            CAS (&T, tail, newTail)
            return 
        } else {
            // help other enqueue operations
            CAS(&T, tail, tail.N)
        }
    }
}
```

Если убрать ветку `else`, то алгоритм станет блокирующимся -- новые потоки будут ждать `CAS(&T ...)`.

Удаление реализуется так же, как в стеке.
