# Лекция 7.1. Мониторы и ожидание

## Ожидание

Пусть операция над объектом это функция \\(f(S, P) = (S', R)\\). Раньше операции были всюду определены на паре \\((S, P)\\), то есть если операцию нельзя выполнить, то это исключение. Однако в общем случае операции могут быть частично определены, т.е. операция не может завершиться и ждет.

Например, очередь ограниченного размера с ожиданием:
- `put(item)` - кладет элемент в очередь, _если есть место_ (иначе ждет)
- `take(): item` - забирает элемент из очереди, _если очередь не пустая_ (иначе ждет)

Это часто происходит в паттерне producer-cosumer (привет, ГК).

Пусть в исполнении может не быть \\(res(A)\\). Тогда будем называть исполнение линеаризуемым, если для незавершенных операций можно:
- или добавить ответы,
- или выкинуть их из исполнения

Так, чтобы получилось допустимое последовательное исполнение.

## Мониторы

Это mutex + условные переменные.

В jvm у каждого объекта есть монитор с одной условной переменной, `wait`, `notify`, `notifyAll` работают с ней.

### Циклическая очередь на массиве

```java
public class BlockingQueue<T> {
    private final T[] items; // элементы
    private final int n; // == items.length
    private int head; // голова
    private int tail; // хвост
    
    public synchronized int size() {
        return (tail - head + n) % n;
    }
    
    // не ждущий
    public synchronized T poll() {
        if (head == tail) return null;
        T result = items[head];
        items[head] = null;
        head = (head + 1) % n;
        return result;
    }
    
    // ждущий
    public synchronized T take() throws InterruptedException /* позже поговорим */ {
        while (head == tail) wait(); // ждем
        // wait может сам по себе проснуться, поэтому его надо делать в цикле
        T result = items[head];
        items[head] = null;
        head = (head + 1) % n;
        return result;
    }
}
```

Метод `wait` - часть монитора, освобождает блокировку и ждёт сигнала о пробуждении. Сингал посылается через `notify` и `notifyAll`. Оба могут быть использованы только в критической секции.

```java
// не ждущий
public synchronized boolean offer(T item) {
    int next = (tail + 1) % n;
    if (next == head) return false;
    items[tail] = item;
    if (head == tail) notifyAll();
    tail = next;
    return true;
}

public synchronized void put(T item) throws Inter...Ex... {
    while (true) { // пока не подходящее состояние
        int next = (tail + 1) % n;
        if (next == head) { wait(); continue; }
        items[tail] = item;
        if (head == tail) notifyAll();
        tail = next;
        return;
    }
}
```

Нам не важно, где в коде вызван `notifyAll`.

`notify` более точечный, потому что можно пробуждать только один поток. В нашем примере это не сработает, т.к. `put` может пробудить другой `put` поток, а нам нужно будить `take` потоки. Однако в `ReentrantLock` можно заводить условные переменные через `lock.newCondition()`. На других языках условные переменные используются только так.

```kotlin
class BlockingQueue<T>(private val n: Int) {
    private val items = arrayOfNulls<Any>(n)
    private var head = 0
    private var tail = 0

    private val lock = ReentrantLock()
    private val notEmpty = lock.newCondition()
    private val notFull = lock.newCondition()
}

fun take(): T = lock.withLock {
    while (head == tail) notEmpty.await() // ждем
    val result = items[head] as T
    items[head] = null
    // ТУТ БАГ!
    if ((tail + 1) % n == head) notFull.signal() // была полна
    head = (head + 1) % n
    result // вернули из withLock
}

fun put(item: T): Unit = lock.withLock {
    while (true) { // пока не подходящее состояние
        val next = (tail + 1) % n
        if (next == head) { notFull.await(); continue }
        items[tail] = item
        // ТУТ БАГ!
        notEmpty.signal() // надо посылать один сигнал
        tail = next
        return@withLock
    }
}
```

Два сигнала от разных `take` могут сигнальнуть одному и тому же `put`. Нам не гарантировано, что сигнальнутый поток сразу проснется, поэтому он может проснуться после обоих сигналов и тогда другой `put` не проснется, а должен бы.

Простой способ починить: слать сигнал без условия. В худшем случае сигнал быстро отработает, т.к. будить некого.

Переключение контекста (разбудить поток) — очень дорого.

## Interrupt

У каждого потока есть флаг `interrupted`. Он выставляется методом `Thread.interrupt`, его проверяют методы `wait`/`await` и если он выставлен, то сбрасывают его и кидают `InterruptedException`. Таким образом можно кооперативно прекращать ожидание.

Если мы не знаем, что делать с `InterruptedException`, то надо писать так:

```java
public T takeOrNull() {
    try {
        return take();
    } catch (InterruptedException e) {
        // перевыставим флаг interrupted
        Thread.currentThread().interrupt();
        return null;
    }
}
```

```java
public class DoSomethingThread<T> extends Thread {
    private final BlockingQueue<T> queue; // задачи
    private volatile boolean closed; // флаг останова

    public void close() {
        closed = true; // ставим флаг останова (сначала!)
        interrupt(); // чтобы прервать ожидания
    }

    @Override
    public void run() {
        try {
            while (!closed) {
            T item = queue.take();
            doSomething(item);
            }
        } catch (InterruptedException e) {
            // а вот здесь можем проигнорировать -- уже выходим
        }
    }
}
```

### Обновляемое значение

Реализация с блокировкой:
```kotlin
class DataHolder<T> {
    private var value: T? = null
    private val lock = ReentrantLock()

    fun update(item: T) = lock.withLock {
        value = item
    }

    fun remove(): T? = lock.withLock {
        value.also { value = null }
    }
}
```

Реализация с ожиданием:
```kotlin
private val updated = lock.newCondition()

fun take(): T = lock.withLock {
    while (value == null) updated.await()
    value!!.also { value = null }
}

fun update(item: T) = lock.withLock {
    value = item
    updated.signal()
}
```

Реализация без блокировки, но не ждущими методами:
```kotlin
class DataHolder<T> {
    private val v = atomic<T?>(null)

    fun update(item: T) {
        v.value = item // volatile write
    }

    fun remove(): T? {
        v.loop { cur ->
            if (cur == null) return null
            if (v.compareAndSet(cur, null)) return cur
        }
    }
}
```

Ожидание без блокировки через `park`:
```kotlin
class TakerThread<T> : Thread() {
// ...

    fun take(): T {
        assert(Thread.currentThread() == this)
        v.loop { cur ->
            if (cur == null) {
                LockSupport.park()
                if (interrupted()) // ручная проверка флага
                    throw InterruptedException()
                return@loop // continue loop
            }
            if (v.compareAndSet(cur, null)) return cur
        }
    }
    
    fun update(item: T) {
        v.value = item // volatile write
        LockSupport.unpark(this)
    }
}
```

Поток будит только сам себя. Кроме того, `unpark` будит не только уже спящий поток, но и поток который уснет, т.е. `park` будет no-op.

Если мы хотим сделать ожидание из многих потоков, то надо использовать `AbstractQueuedSynchronizer`, правда он вообще-то предназначен для написания блокировок .

```kotlin
inner class Sync : AbstractQueuedSynchronizer() {
    override fun tryAcquire(arg: Int): Boolean {
        val cur = v.value ?: return false
        if (!v.compareAndSet(cur, null)) return false
        // надо как-то вернуть значение отсюда, будем держать в поле
        results[arg] = cur
        return true
    }

    // всегда "освобождаем" -- будим следующего
    override fun tryRelease(arg: Int): Boolean = true
}

private val sync = Sync()

fun update(item: T) {
    v.value = item // volatile write
    sync.release(0) // шлем сигнал
}

fun take(): T {
    val arg = reserveResultsSlot() // приходится крутиться
    sync.acquireInterruptibly(arg) // ждет внутри
    // нужна перепроверка чтобы не потерять unpark
    if (v.value != null) sync.release(0)
    return releaseResultsSlot(arg)
}
```
