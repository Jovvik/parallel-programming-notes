# Лекция 6. FAA-Based Queue & Flat Combining

### FAA

FAA это `Fetch-And-Add(addr, delta)`, атомарно увеличивает значение на `delta` и возвращает старое значение. Это лучше масштабируется, чем `CAS` цикл, так как операция всегда успешна.

### Obstruction-free queue

Пусть у нас есть бесконечный массив с двумя указателями `enqIdx`, `deqIndx`. Что делать, если `dequeue` не увидел никакую запись? Тогда пометим ячейку как сломанную и обе операции начнутся заново.

```kotlin
fun enqueue(x: T) = while (true) {
    val enqIdx = FAA(&enqIdx, 1)
    if (CAS(&data[enqIdx], null, x))
    return
}

fun dequeue() = while (true) {
    if (isEmpty()) return null
    val deqIdx = FAA(&deqIdx, 1)
    val res = SWAP(&data[deqIdx], BROKEN)
    if (res == null) continue
    return res
}

fun isEmpty(): Boolean = deqIdx >= enqIdx
```

Это не lock-free, но на практике он почти lock-free. Кроме того, lock-free алгоритмы на практике почти всегда являются wait-free в силу честности планировщиков ОС.

В реальности у нас не бесконечный массив. Поэтому будем хранить Michal-Scott очередь сегментов.

```kotlin
fun enqueue(x: T) = while (true) {
    val tail = this.tail
    val enqIdx = FAA(&tail.enqIdx, 1)
    if (enqIdx >= NODE_SIZE) {
        // try to insert new node with “x”
    } else {
        if (CAS(&tail.data[enqIdx], null, x))
            return
    }
}

fun dequeue(): T = while (true) {
    val head = this.head
    val deqIdx = FAA(&head.deqIdx, 1)
    if (deqIdx >= NODE_SIZE) {
        val headNext = head.next ?: return null
        CAS(&this.head, head, headNext)
        continue
    }
    val res = SWAP(&head.data[deqIdx], BROKEN)
    if (res == null) continue
    return res
}
```

Тогда алгоритм lock-free.

### Flat combining

Идея: будем выполнять операции последовательно, структура данных защищена блокировкой. Поток, который держит блокировку — комбайнер, он выполняет как свою, так и чужие операции. Таким образом, мы не проигрываем по кешу и не так часто берем блокировку.

Это делается через массив, в который (случайно) кладутся дескрипторы операций. Держащий блокировку поток обходит весь массив и выполняет все операции, которые видит. Массив очевидно атомарный.

Очередь на flat combining куда быстрее, чем michael-scott. Однако FAA еще быстрее. В целом flat combining полезно, когда грубая/тонкая блокировка слишком медленно, а flat combining хватает. Бывает, что flat combining быстрее чем lock-free (см. очередь Michael-Scott).

Лок для flat combining очень простой: просто атомарный бул:

```kotlin
var locked = false

fun tryLock() = CAS(&lock, false, true)

fun unlock() { locked = false }
```
