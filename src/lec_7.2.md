# Лекция 7.2. Сложные блокировки

Мы раньше строили таблицу конфликтов для регистров, но мы можем так же делать и для своих структур. Например, для стека:

|        | `size` | `push` | `pop` |
| ------ | ------ | ------ | ----- |
| `size` |        | x      | x     |
| `push` | x      | x      | x     |
| `pop`  | x      | x      | x     |

Если у нас грубая блокировка, то мы исключили `size`-`size`, но это не нужно. Чтобы не блокать то, что не нужно, придуманы read-write lock'и, которые имеют два режима: Read (Shared) и Write (Exclusive).

|     | R   | W   |
| --- | --- | --- |
| R   |     | x   |
| W   | x   | x   |

Пример:
```kotlin
private val lock = ReentrantReadWriteLock()

val size: Int get() = lock.readLock().withLock {
    top
}

fun push(item: T) = lock.writeLock().withLock {
    data[top++] = item
}

fun pop(): T = lock.writeLock().withLock {
    data[--top]
}
```

Хотелось бы сделать что-то такое, но это deadlock потока с самим собой:
```kotlin
val lock = ReentrantReadWriteLock()
lock.readLock().withLock {
    println("reading")
    lock.writeLock().withLock {
        println("writing")
    }
}
```

Казалось бы, можно придумать `UpgradableReadWriteLock` со следующей матрицей совместимости:

|           | \\(R_P\\) | \\(R_Q\\) | \\(W_P\\) | \\(W_Q\\) |
| --------- | --------- | --------- | --------- | --------- |
| \\(R_P\\) |           |           |           | x         |
| \\(R_Q\\) |           |           | x         |           |
| \\(W_P\\) |           | x         |           | x         |
| \\(W_Q\\) | x         |           | x         |           |

Однако если несколько потоков хотят `upgrade`нуться, то есть опасность deadlockа. Тогда создадим состояние "Intent to write", которое будет эксклюзивным:

|     | R   | IW  | W   |
| --- | --- | --- | --- |
| R   |     |     | x   |
| IW  |     | x   | x   |
| W   | x   | x   | x   |

Напишем такой хитрый лок.

```kotlin
private const val R_BIT = 1
private const val R_MASK = (1 shl 30) - 1
private const val IW_BIT = 1 shl 30
private const val W_BIT = 1 shl 31

class Sync : AbstractQueuedSynchronizer() {
    override fun tryAcquireShared(arg: Int): Int { // R или IW
        while (true) {
            val state = this.state
            if (state and W_BIT != 0) // если кто-то W, то нельзя
                return -1
            // если кто-то еще IW и мы хотим IW, то нельзя
            if (arg == IW_BIT && state and IW_BIT != 0) 
                return -1
            val update = state + arg
            if (compareAndSetState(state, update))
                return 1
        }
    }
    
    override fun tryReleaseShared(arg: Int): Boolean {
        while (true) {
            val state = this.state
            val update = state - arg
            if (compareAndSetState(state, update))
                return update and (R_MASK or IW_BIT) == 0
        }
    }
    
    override fun tryAcquire(arg: Int): Boolean { // тривиально
        while (true) {
            val state = this.state
            if (state != 0)
                return false
            if (compareAndSetState(state, state or W_BIT))
                return true
        }
    }
    
    override fun tryRelease(arg: Int): Boolean {
        while (true) {
            val state = this.state
            val update = state and W_BIT.inv()
            if (compareAndSetState(state, update))
                return true
        }
    }
}
```

Ещё нужно в `ThreadLocal` состоянии держать сколько локов каждого типа мы держим, чтобы если мы держим лок, не брать его физически.
