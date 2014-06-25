# Алгоритм хеширования и коллизии

В этом разделе мы подробнее изучим самые плохие случаи коллизий и некоторые особенности хэш-функции, которая используетсяв PHP. Хотя знание этой информации и необязательно для использования API хэш-таблиц, но она даст вам более полное представление о структуре хэш-таблицы и её ограничения.

##Анализ коллизий

Для того чтобы упростить анализ коллизий давайте сначала разработаем вспомогатенльную функцию `array_collision_info()`, которая принимает массив и показывает какие ключи преобразуются в какие индексы. Чтобы сделать это мы пройдемся по `arBuckets` и для каждого индекса создадим массив содержащий инвормацию о всех бакетах этого индекса:
```c
PHP_FUNCTION(array_collision_info) {
    HashTable *hash;
    zend_uint i;

    if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_DC, "h", &hash) == FAILURE) {
        return;
    }

    array_init(return_value);

    /* Empty hashtables may not yet be initialized */
    if (hash->nNumOfElements == 0) {
        return;
    }

    for (i = 0; i < hash->nTableSize; ++i) {
        /* Create array of elements at this nIndex */
        zval *elements;
        Bucket *bucket;

        MAKE_STD_ZVAL(elements);
        array_init(elements);
        add_next_index_zval(return_value, elements);

        bucket = hash->arBuckets[i];
        while (bucket != NULL) {
            zval *element;

            MAKE_STD_ZVAL(element);
            array_init(element);
            add_next_index_zval(elements, element);

            add_assoc_long(element, "hash", bucket->h);

            if (bucket->nKeyLength == 0) {
                add_assoc_long(element, "key", bucket->h);
            } else {
                add_assoc_stringl(
                    element, "key", (char *) bucket->arKey, bucket->nKeyLength - 1, 1
                );
            }

            {
                zval **data = (zval **) bucket->pData;
                Z_ADDREF_PP(data);
                add_assoc_zval(element, "value", *data);
            }

            bucket = bucket->pNext;
        }
    }
}
```
Этот код также является хорошим примером `add_` функций из предыдущего раздела. Давайте испытаем функцию:
```
var_dump(array_collision_info([2 => 0, 5 => 1, 10 => 2]));

// Output (reformatted a bit):

array(8) {
  [0] => array(0) {}
  [1] => array(0) {}
  [2] => array(2) {
    [0] => array(3) {
      ["hash"]  => int(10)
      ["key"]   => int(10)
      ["value"] => int(2)
    }
    [1] => array(3) {
      ["hash"]  => int(2)
      ["key"]   => int(2)
      ["value"] => int(0)
    }
  }
  [3] => array(0) {}
  [4] => array(0) {}
  [5] => array(1) {
    [0] => array(3) {
      ["hash"]  => int(5)
      ["key"]   => int(5)
      ["value"] => int(1)
    }
  }
  [6] => array(0) {}
  [7] => array(0) {}
}
```
Вот какие выводы мы можем сделать (большинство из них мы уже обсуждали:

* результирующий массив содержит 8 элементов, несмотря на то, что мы вставили всего 4. Это связано с тем, что 8 это размер таблицы по умолчанию.
* Для целых чисел хэш и ключ всегда совпадают.
* Несмотря на то, что все хэши отличаются мы все равно имеем коллизию для `nIndex == 2` так как 2 % 8 равно 2 и 10 % 8 тоже равно 2.
* Связанный список элементов с одинаковыми индексами содержит элементы в обратном порядке (так как это самый простой способ реализации).

##Коллизии индексов

Теперь наша цель создать такой сценарий, при котором все хэши ключей будут конфликтовать друг с другом. Есть два способа сделать это и мы начнем с простейшего. Вместо того чтобы создавать коллизии в хэш-функции мы создадим коллизии в индексах (хэш которых вычисляется как деление по модулю значения хеша на размер таблицы).

Для целочисленных ключей это просто, так как к ним не применяется операция хэширования. Индекс будет равен `key % nTableSize`. Поиск коллизий для такого выражения тривиален, так как все ключи являющиеся множителями для размера таблицы будут конфликтовать друг с другом. Например, если размер таблицы равен 8, то конфликтовать будут индексы 0 % 8 = 0, 8 % 8 = 0, 16 % 8 = 0, 24 % 8 = 0, и т.д.

Ниже приведен PHP-скрипт демонстрирующий этот сценарий:
```c
<?php

$size = pow(2, 16); // any power of 2 will do

$startTime = microtime(true);

// Insert keys [0, $size, 2 * $size, 3 * $size, ..., ($size - 1) * $size]

$array = array();
for ($key = 0, $maxKey = ($size - 1) * $size; $key <= $maxKey; $key += $size) {
    $array[$key] = 0;
}

$endTime = microtime(true);

printf("Inserted %d elements in %.2f seconds\n", $size, $endTime - $startTime);
printf("There are %d collisions at index 0\n", count(array_collision_info($array)[0]));
```
Вот результат полученный мной (вы можете получить другой результат, но у него будет тот же порядок):
```
Inserted 65536 elements in 34.05 seconds
There are 65536 collisions at index 0
```
Конечно же 30 секунд на вставку элементов это очень долго. Что же произошло? Так как мы смоделировали ситуацию, в которой хэши всех ключей конфликтуют, производительность операций вставки упала с O(1) до O(n). То есть на каждую вставку PHP должен обойти весь связанный список элементов с одинаковым индексом и проверить существует ли элемент с таким ключом. Обычно это не проблема, если связанный список элементов с одним индексом содержит 1 или 2 бакета. В худшем случае все элементы могут быть в этом списке.

В таком случае PHP выполняет n вставок за время O(n), что в итоге дает время выполнения O(n^2). Таким образом, вместо выполнения 2^16 операций должно быть выполнено 2^32.

##Коллизии хэшей

Теперь, когда мы успешно смоделировали худший сценарий коллизий для целочиселнных ключей, давайте продлелаем тоже самое для строковых ключей. Сначала мы взглянем на то как работает хэш-функция в PHP:
```c
static inline ulong zend_inline_hash_func(const char *arKey, uint nKeyLength)
{
    register ulong hash = 5381;

    /* variant with the hash unrolled eight times */
    for (; nKeyLength >= 8; nKeyLength -= 8) {
        hash = ((hash << 5) + hash) + *arKey++;
        hash = ((hash << 5) + hash) + *arKey++;
        hash = ((hash << 5) + hash) + *arKey++;
        hash = ((hash << 5) + hash) + *arKey++;
        hash = ((hash << 5) + hash) + *arKey++;
        hash = ((hash << 5) + hash) + *arKey++;
        hash = ((hash << 5) + hash) + *arKey++;
        hash = ((hash << 5) + hash) + *arKey++;
    }
    switch (nKeyLength) {
        case 7: hash = ((hash << 5) + hash) + *arKey++; /* fallthrough... */
        case 6: hash = ((hash << 5) + hash) + *arKey++; /* fallthrough... */
        case 5: hash = ((hash << 5) + hash) + *arKey++; /* fallthrough... */
        case 4: hash = ((hash << 5) + hash) + *arKey++; /* fallthrough... */
        case 3: hash = ((hash << 5) + hash) + *arKey++; /* fallthrough... */
        case 2: hash = ((hash << 5) + hash) + *arKey++; /* fallthrough... */
        case 1: hash = ((hash << 5) + hash) + *arKey++; break;
        case 0: break;
        EMPTY_SWITCH_DEFAULT_CASE()
    }
    return hash;
}
```
Если убрать ручное развертывание цикла, то функция будет выглядеть так:
```c
static inline ulong zend_inline_hash_func(const char *arKey, uint nKeyLength)
{
    register ulong hash = 5381;

    for (uint i = 0; i < nKeyLength; ++i) {
        hash = ((hash << 5) + hash) + arKey[i];
    }

    return hash;
}
```
Выражение `hash << 5 + hash` равнозначно выражению `hash * 32 + hash` или просто `hash * 33`. Зная это мы можем еще упростить функцию:
```c
static inline ulong zend_inline_hash_func(const char *arKey, uint nKeyLength)
{
    register ulong hash = 5381;

    for (uint i = 0; i < nKeyLength; ++i) {
        hash = hash * 33 + arKey[i];
    }

    return hash;
}
```
Эта хэш-функция называется *DJBX33A*, название происходит от “Daniel J. Bernstein, Times 33 with Addition”. Это одна из самых простых (и в то же время быстрых) хэширующих функций для строк.

Благодаря простоте этой функции поиск коллизий не составит труда. Мы начнем с поиска коллизий для двухбуквенных строк, то есть мы хотим найти две такие строки `ab` и `cd`, которые имеют одинаковый хеш:
```c
    hash(ab) = hash(cd)
<=> (5381 * 33 + a) * 33 + b = (5381 * 33 + c) * 33 + d
<=> a * 33 + b = c * 33 + d
<=> c = a + n
    d = b - 33 * n
    where n is an integer
```
Исходя из этого видно, что для того чтобы получить коллизию для двухбуквенных строк нужно увеличить первую букву на единицу и уменьшить вторую на 33. Используя эту технику мы можем создать группу из 8 строк с одинаковыми хэшами:
```
<?php
$array = [
    "E" . chr(122)  => 0,
    "F" . chr(89)   => 1,
    "G" . chr(56)   => 2,
    "H" . chr(23)   => 3,
    "I" . chr(-10)  => 4,
    "J" . chr(-43)  => 5,
    "K" . chr(-76)  => 6,
    "L" . chr(-109) => 7,
];

var_dump(array_collision_info($array));
```
Этот код покажет, что все ключи имеют хэш `193456164`:
```
array(8) {
  [0] => array(0) {}
  [1] => array(0) {}
  [2] => array(0) {}
  [3] => array(0) {}
  [4] => array(8) {
    [0] => array(3) {
      ["hash"]  => int(193456164)
      ["key"]   => string(2) "L\x93"
      ["value"] => int(7)
    }
    [1] => array(3) {
      ["hash"]  => int(193456164)
      ["key"]   => string(2) "K´"
      ["value"] => int(6)
    }
    [2] => array(3) {
      ["hash"]  => int(193456164)
      ["key"]   => string(2) "JÕ"
      ["value"] => int(5)
    }
    [3] => array(3) {
      ["hash"]  => int(193456164)
      ["key"]   => string(2) "Iö"
      ["value"] => int(4)
    }
    [4] => array(3) {
      ["hash"]  => int(193456164)
      ["key"]   => string(2) "H\x17"
      ["value"] => int(3)
    }
    [5] => array(3) {
      ["hash"]  => int(193456164)
      ["key"]   => string(2) "G8"
      ["value"] => int(2)
    }
    [6] => array(3) {
      ["hash"]  => int(193456164)
      ["key"]   => string(2) "FY"
      ["value"] => int(1)
    }
    [7] => array(3) {
      ["hash"]  => int(193456164)
      ["key"]   => string(2) "Ez"
      ["value"] => int(0)
    }
  }
  [5] => array(0) {}
  [6] => array(0) {}
  [7] => array(0) {}
}
```
Получив группу строк с одинаковыми хэшами дальнейший поиск коллизий становится очень простым. Для этого достаточно воспользоватьсясвойством DJBX33A: если две строки с одинаковой длиной `$str1` и `$str2` имеют одинаковый хэш, то строки `$prefix.$str1.$postfix` и `$prefix.$str2.$postfix` также будут иметь одинаковый хэш. Легко убедиться что это действительно так:
```
  hash(prefix . str1 . postfix)
= hash(prefix) * 33^a + hash(str1) * 33^b + hash(postfix)
= hash(prefix) * 33^a + hash(str2) * 33^b + hash(postfix)
= hash(prefix . str2 . postfix)

  where a = strlen(str1 . postfix) and b = strlen(postfix)
```
Таким образом, если `Ez` и `FY` имеют одинаковый хэш, то и `abcEzefg` и `abcFYefg` также будут иметь одинаковый хэш. Это является причиной почему мы можем игнорировать нулевой байт в конце ключа: его наличие даст другой хэш, но если хэши ключей без нулевого байта совпадают, то они будут совпадать и для тех же ключей с нулевым байтом.

Используя это свойство мы можем создать большой набор ключей с конфликтущими хэшами взяв известный набор конфилктующих ключей и объединив их разными способами. То есть если мы знаем, что `Ez` и `FY` имеют конфликтующие хэши, то хэши будут конфликтовать и для `EzEzEz`, `EzEzFY`, `EzFYEz`, `EzFYFY`, `FYEzEz`, `FYEzFY`, `FYFYEz` и `FYFYFY`. Используя этот метод можно создать какой угодно большой набор коллизий.
