# Лекция 8. Многопоточные хеш-таблицы

Мы будем использовать открытую адресацию ради локальности по кешу. 

Без перестроения таблицы чтения тривиальны, но что делать когда мы перестраиваем таблицу? Будем читать из старого массива и атомарно менять ссылку на массив, когда перестроение закончено.

![](img/cell.png)

Когда мы перемещаем ячейку, мы помечаем её как перемещенную и запрещаем ее изменять.

![](img/cell2.png)

Для конкретных use case'ов можно оптимизировать нашу структуру.
