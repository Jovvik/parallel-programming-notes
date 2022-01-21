# Лекция 9. Dual Data Structures

Часто встречается паттерн producer-consumer, где n клиентов вбрасывают в пул задачи и m workerов их выполняют. Типичный способ коммуникации клиентов и workerов - synchronous queue:

```kotlin
interface SynchronousQueue<E> {
    suspend fun send(element: E)
    suspend fun receive(): E
}
```

Как мы используем такую очередь?

Клиенты шлют задачи:
```kotlin
val task = Task(...)
tasks.send(task)
```

Worker'ы получают задачи в бесконечном цикле:
```kotlin
while (true) {
    val task = tasks.receive() // спим пока задач нет
    processTask(task)
}
```

Мы часто ложимся спать и просыпаемся, это медленно. Поэтому мы хотим использовать корутины — это легковесные потоки. В корутинах общение происходит через каналы, что более безопасно, чем общая память.

API корутин примерно такое:
```kotlin
class Coroutine {
    var element: Any? // то что отправляем
    ...
}

fun curCroruntine(): Coroutine

suspend fun suspend(c: Coroutine)
fun resume(c: Coroutine)
```

Реализуем канал, пока что не думая про гонки:
```kotlin
val senders = Queue<Coroutine>()
val receivers = Queue<Coroutine>()

suspend fun send(element: T) {
    if (receivers.isEmpty()) {
        val curCor = curCoroutine()
        curCor.element = element
        senders.enqueue(curCor)
        suspend(curCor)
    } else {
        val r = receivers.dequeue()
        r.element = element
        resume(r)
    }
}

suspend fun receive(): T {
    if (senders.isEmpty()) {
        val curCor = curCoroutine()
        receivers.enqueue(curCor)
        suspend(curCor)
        return curCor.element
    } else {
        val s = senders.dequeue()
        val res = s.element
        resume(s)
        return res
    }
}
```

В golang receive и send заворачиваются в грубую блокировку. Это не очень быстро. В java все быстрее за счет идеи, что: либо одна очередь пустая, либо другая, либо обе. Тогда будем держать только одну очередь (Майкла-Скотта), в которой храним пару (корутина, элемент (который надо отправить) (или некоторая константа если это `receive`)).

```kotlin
fun send(x) {
    t := TAIL
    h := HEAD
    if (t == h || t.isSender()) {
        enqueueAndSuspend(t, x)
    } else {
        dequeAndResume(h)
    }
}
```

Это не линеаризуемо в силу засыпания, потому что операции ждут друг друга. Поэтому мы разбиваем `receive` на две части:
- `request`, которая регистрирует поток как ждущий
- `follow-up`, которая работает после получения данных

Тогда у нас есть линеаризуемость.

### FAA

Каналы похожи на очереди. Мы их можем ускорить с помощью `Fetch-and-add`, с двумя указателями.

```kotlin
suspend fun send(x: T) {
    val s = FAA(&sendIdx, +1)
    if s < receiveIdx:
        // try to put f into w[s]
        // or resume the receiver
    else:
        // suspend in w[s]
}

suspend fun receive(): T {
    val r = FAA(&receiveIdx, +1)
    if r < sendIdx:
        // try to retrieve either
        // value or sender from w[r]
    else:
        // suspend in w[r]
}
```
