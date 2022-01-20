# Лекция 3.1 Практические построения на списках

## Множество на односвязном списке

Это казалось бы игрушечная структура данных, но она используется в очень полезном skip listе.

Пусть в списке у нас есть граничные элементы \\(-\infty, +\infty\\).

- Элементы упорядочены по возрастанию
- Мы ищем окно (`cur`, `next`), такое что `cur.key` \\(<\\) `k` \\(\\leq\\) `next.key` и `cur.N = next`
- Если мы ищем элемент, то он будет в `next` (или его не будет)
- Новый элемент будем добавлять между `cur` и `next`.

Псевдокод:
```kotlin
class Node(var N: Node, val key: Int)

val head = Node(−∞, Node(∞, null))

fun findWindow(key): (Node, Node) {
    cur := head
    next := cur.N
    while (next.key < key):
        cur = next
        next = cur.N
    return (cur, next)
}

fun contains(key): Boolean {
    (cur, next) := findWindow(key)
    return next.key == key
}

fun add(key) {
    (cur, next) := findWindow(key)
    if (next.key != key)
        cur.N = Node(key, next)
}

fun remove(key) {
    (cur, next) := findWindow(key)
    if (next.key == key)
        cur.N = next.N
}
```

Проблема, удаление последовательных вершин в различных потоках может не сработать.

`a -> b -> c -> d` станет `a -> c -> d` и `b -> d`, в итоге `c` не удалено.

Можно навесить грубую синхронизацию, но это неэффективно.

### Тонкая синхронизация

Можно навесить лок на каждую вершину. Тогда поток будет держать блокировку на `cur` и `next`. В коде это будет выглядеть так:

```kotlin
fun findWindow(key): (Node, Node) {
    cur := head; cur.lock()
    next := cur.N; next.lock()
    while (next.key < key):
        cur.unlock(); cur = next
        next = cur.N; next.lock()
    return (cur, next)
}

fun contains(key): Boolean {
    (cur, next) := findWindow(key)
    cur.unlock(); next.unlock()
    return next.key == key
}
```

### Оптимистичная синхронизация

1. Найти окно без синхронизации.
2. Взять блокировки на `cur` и `next`.
3. Проверить, что `cur.N == next`.
4. Проверить, что `cur` не удален.
5. Выполнить операцию.
6. При ошибке попробовать заново.

При проверке, что `cur` не удален, мы держа блокировку на него, ищем его еще раз в линию.

```kotlin
class Node(@Volatile var N: Node, val key: Int)

fun contains(key): Boolean {
    while (true) {
        (cur, next) := findWindow(key)
        cur.lock(); next.lock()
        if (!validate(cur, next))
            cur.unlock(); next.unlock(); continue
        return next.key == key
    }
}

fun validate(cur, next): Boolean {
    var node = head
    while(node.key < cur.key):
    node = node.N
    return (cur, next) == (node, node.N)
}
```

### Ленивая синхронизация

Будем лениво удалять. В `Node` добавляем `removed: Boolean`. Удаление будет происходить в две фазы:
1. `node.removed = true` - логическое удаление
2. Физическое удаление

Тогда валидация тривиальна:

```kotlin
fun validate(cur, next) = !cur.removed && !next.removed && cur.N == next
```

### Неблокирующая синхронизация

Т.к. `N` `volatile`, то мы увидели состояние памяти на момент записи в `N`. Поэтому мы можем не брать блокировку при поиске. Тогда:

```kotlin
fun contains(key): Boolean {
    (cur, next) := findWindow(key)
    return next.key == key
}
```

Наивная запись на `CAS` опять не работает. `a -> b -> c -> d` станет `a -> c -> d` и `b -> d`, в итоге `c` не удалено. Проблема в том, что мы не знали, что `b` уже удалено. Казалось бы, можно написать двусвязный список, но нет, это очень сложно. Вместо этого объединим `N` и `removed` в одну переменную, пару `(N, removed)` и таким образом запретим делать изменения на `N`, если нода удалена. В java это `AtomicMarkableReference`, реализовано через обертку.

```kotlin
fun findWindow(key): (Node, Node) {
    retry: while(true):
        var cur = head, next = cur.N
        boolean[] removed = new boolean[1]
        while (next.key < key):
            val node = next.N.get(removed)
            if (removed[0]):
                // удалим физически
                if (!cur.N.CAS(next, node, false, false)):
                    continue retry
                next = node
            else:
                cur = next
                next = cur.N
        // тут еще проверка, что next не удален
        return (cur, next)
}

fun contains(key): Boolean {
    (cur, next) = findWindow(key)
    return next.key == key
}

fun add(key) {
    while(true):
        (cur, next) = findWindow(key)
        if (next.key == key):
            return
        val node = Node(key, next)
        if (cur.N.CAS(next, node, false, false)):
            return
}

fun remove(key) {
    while(true):
        (cur, next) = findWindow(key)
        if (next.key != key)
            return // false
        val node = next.N.getReference();
        if (next.N.CAS(node, node, false, true)):
            // помогаем findWindow удалить физически
            cur.N.CAS(next, node, false, false)
            return // true
}
```
