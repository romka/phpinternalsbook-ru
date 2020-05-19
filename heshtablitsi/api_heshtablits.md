# API хеш-таблиц

В PHP есть два набора API для работы с хеш-таблицами: первый набор это низкоуровневый `zend_hash` API, он рассмотрен в этом разделе. Второй набор — это API для работы с массивами, который предоставляет высокоуровненвые функции для выполнения распространенных операций. Второй набор рассмотрен в следующем разделе.

##Создание и удаление хеш-таблиц

Память для хеш-таблиц выделяется с использованием `ALLOC_HASHTABLE()`, инициализация выполняется при помощи `zend_hash_init()`:
```c
HashTable *myht;

/* Same as myht = emalloc(sizeof(HashTable)); */
ALLOC_HASHTABLE(myht);

zend_hash_init(myht, 1000000, NULL, NULL, 0);
```
Второй аргумент в `zend_hash_init()` это size hint (подсказка по размеру хеш-таблицы), который определяет как много элементов вы ожидаете иметь в хеш-таблице. В случае когда передано значение 1000000 PHP выделит место под for 2^20 = 1048576 элементов в момент первой вставки. Без такой подсказки PHP сначала выделит место под 8 элементов и затем будет выполнять множество изменений размера хеш-таблицы при вставке новых элементов (сначала до 16, потом до 32, до 64 и т.д). Каждое изменение размера требует чтобы память под `arBuckets` была выделена заново (reallocated), а также требует пересчета всех хешей (“rehash”), что приводит к пересчету коллизий.

Использование size hint помогает избежать ненужных операций изменения размера и улучшает производительность. Но это имеет смысл только для больших хеш-таблиц, для маленьких достаточно установить size hint в 0. Так как обычно минимальный размер таблицы равен 8, то нет разницы каким вы сделаете сайз хинт 0, 2, или 7.

Третий аргумент функции `zend_hash_init()` всегда должен иметь значение `NULL`. Ранее он использовался для указания кастомной хеш-функции, но сейчас этот аргумент больше не поддерживается. Четвертый аргумент — это функция-деструктор, применяемая для хрантимых значений, она имеет такую сигнатуру:
```c
typedef void (*dtor_func_t)(void *pDest);
```
В большинстве случаев используется ` ZVAL_PTR_DTOR` (для хранимых значений типа `zval *`). Это обычный `zval_ptr_dtor()`, но его сигнатура совместима с `dtor_func_t`.

Последний аргумент функции `zend_hash_init()` определяет следует ли использовать persistent allocation. Если вы хотите, чтобы хеш-таблица существовала после завершения запроса этот аргумент должен быть равен `1`. Также существует другая функция инициализации `zend_hash_init_ex()`, она принимает дополнительный аргумент `bApplyProtection` типа boolean. Установка его в 0 позволяет отключить защиту от рекурсии (которая включена по умолчанию). Эта функция используется редко, обычно для внутрениих структур движка (таких как функции или class table).

Хеш-таблица может быть уничтожена с помощью `zend_hash_destroy()` и освобождена с помощью `FREE_HASHTABLE()`:
```c
zend_hash_destroy(myht);

/* Same as efree(myht); */
FREE_HASHTABLE(myht);
```
Функция `zend_hash_destroy()` вызовет деструктор для всех бакетов и освободит их. Пока эта функция выполняется хеш-таблица находится в несогласованном  (inconsistent) состоянии и не может быть использована. Обычно это не проблема, но в некоторых редких случаях (особенно когда деструктор может быть вызван в пользовательском коде) может понадобиться, чтобы хеш-таблица могла быть использована в процессе работы деструктора. В этом случае могут быть использованы функции `zend_hash_graceful_destroy()` и `zend_hash_graceful_reverse_destroy()`. Первая функция удаляет бакеты в порядке, в котором они были вставлены, вторая — в обратном порядке.

Если вам нужно удалить все элементы хеш-таблицы, но не уничтожить саму хеш-таблицу, вы можете использовать функцию `zend_hash_clean()`.

##Числовые ключи

Прежде чем изучать функции использующиеся для вставки, извлечения и удаления числовых ключей в хеш-таблицы, давайте проясним какие аргументы нужны этим функциям.

Вспомним, что член-данных бакета `pData` хранит указатель на реальные данные. Таким образом, если вы храните в хеш-таблице указатели `zval *`, тогда `pData` будет `zval **`. По этой причине вставка в хеш-таблицу потребует передачи `zval **`, несмотря на то, что вы определили тип данных как `zval *`.

Когда вы извлекаете значения из хеш-таблицы вам нужно будет передать в функцию указатель на переменную назначения `pDest`, в которую будет записано значение `pData`. Для того чтобы записать в указатель используя `*pDest = pData` нужен еще один indirection уровень. Итак, если вы записываете в хеш-таблицу `zval *`, то в функцию для извлечения данных вы должны будете передать `zval ***`.

Для примера давайте рассмотрим функцию `zend_hash_index_update()`, которая позволяет вставлять и обновлять целочисленные ключи:
```c
HashTable *myht;
zval *zv;

ALLOC_HASHTABLE(myht);
zend_hash_init(myht, 0, NULL, ZVAL_PTR_DTOR, 0);

MAKE_STD_ZVAL(zv);
ZVAL_STRING(zv, "foo", 1);

/* In PHP: $array[42] = "foo" */
zend_hash_index_update(myht, 42, &zv, sizeof(zval *), NULL);

zend_hash_destroy(myht);
FREE_HASHTABLE(myht);
```
Пример выше вставляет в хеш-таблицу `zval *` содержащий `"foo"` с ключом `42`. The fourth argument specifies the used data type: `sizeof(zval *)`. Третий аргумент, вставляемое значение, должен иметь тип `zval **`.

Последний аргумент может быть использован и для вставки значения, и для извлечения:
```c
zval **zv_dest

zend_hash_index_update(myht, 42, &zv, sizeof(zval *), (void **) &zv_dest);
```
Зачем это может понадобиться? Ведь вы и так знаете значение, которое вы вставили в хеш-таблицу, зачем его извлекать? Помните, что хеш-таблицы работают с *копией* переданного значения. Таким образом, хотя `zval *` сохраненный в хеш-таблице имеет то же значение что и `zv`, но он хранится по другому адресу. Если вы захотите модифицировать это новое значение из хеш-таблицы, то вам нужен указатель на это значение, и это как раз то, что будет записано в `zv_dest`.

При сохранении значений `zval *` последний аргумент функции `zend_hash_index_update` обычно не используется. Однако, при использовании значений, а не указателей, вы часто будете встречаться с паттерном, когда сначала создается временная структура, которая затем вставляется в хеш-таблицу и значение указателя назначения (`**zv_dest`) используется для дальнейшей работы (так как изменения во временной структуре не будут затрагивать значение в хеш-таблице).

Часто вам не нужно вставлять значение с конкретным индексом, достаточно добавить его в конец хеш-таблицы. Это можно сделать с помощью функции `zend_hash_next_index_insert()`:
```c
if (zend_hash_next_index_insert(myht, &zv, sizeof(zval *), NULL) == SUCCESS) {
    Z_ADDREF_P(zv);
}
```

Эта функция вставляет `zv` со следующим доступным целочисленным ключом. Учтите, что в отличии от `zend_hash_index_update()` эта функция может завершиться ошибкой, по этому вам нужно проверять возвращаемое ею значение `SUCCESS`/`FAILURE`.

Чтобы увидеть как ошибка может случиться рассмотрим пример:
```C
zend_hash_index_update(myht, LONG_MAX, &zv, sizeof(zval *), NULL);

php_printf("Next \"free\" key: %ld\n", zend_hash_next_free_element(myht));
if (zend_hash_next_index_insert(myht, &zv, sizeof(zval *), NULL) == FAILURE) {
    php_printf("next_index_insert failed\n");
}
php_printf("Number of elements in hashtable: %ld\n", zend_hash_num_elements(myht));
```
Этот код выведет следующее:
```
Next "free" key: 2147483647 [or 9223372036854775807 on 64 bit]
next_index_insert failed
Number of elements in hashtable: 1
```
Что здесь происходит? Первое значение вставляется с ключом `LONG_MAX`. В этом случае следующим целочисленным ключом должен быть `LONG_MAX + 1`, который выходит за пределы `LONG_MIN`. Такое переполнение недопустимо, PHP делает проверку и, при необходимости, оставляет `nNextFreeElement` равным `LONG_MAX`. Когда `zend_hash_next_index_insert()` запустится, то он попробует вставить значение с ключом `LONG_MAX`, но этот ключ уже занят, по этому функция вернет ошибку.

Последний пример кода использует две новых для нас функции. Одна из них возвращает следующий свободный целочисленный ключ (который, как мы убедились, не обязательно будет свободным), вторая возвращает число элементов в хеш-таблице. Функция `zend_hash_num_elements()` используется довольно часто.

Назначение следующих трех функций должно быть очевидно: `zend_hash_index_find()` возвращает индекс по хешу, `zend_hash_index_exists()` проверяет существует ли индекс без извлечения значения и `zend_hash_index_del()` удаляет элемент. Пример использования функций:
```c
zval **zv_dest;

if (zend_hash_index_exists(myht, 42)) {
    php_printf("Index 42 exists\n");
} else {
    php_printf("Index 42 doesn't exist\n");
}

if (zend_hash_index_find(myht, 42, (void **) &zv_dest) == SUCCESS) {
    php_printf("Fetched value of index 42 into zv_dest\n");
} else {
    php_printf("Couldn't fetch value of index 42 as it doesn't exist :(\n");
}

if (zend_hash_index_del(myht, 42) == SUCCESS) {
    php_printf("Removed value at index 42\n");
} else {
    php_printf("Couldn't remove value at index 42 as it doesn't exist :(\n");
}
```
`zend_hash_index_exists()` возвращает `1` если ключ существует или `0` если не существует. Функции поиска и удаления возвращают `SUCCESS` если значение существует и `FAILURE` если не существует.

##Строкове ключи

Работа со строковыми ключами очень похожа на работу с целочисленными. Основное отличие состоит в том, что в именах функций отсутствует ключевое слово `index`. Эти функции принимают на вход строковый ключ и его длину, а не числовой индекс.

Важно понимать, что а данном контексте означает "длина строки": в API хеш-таблиц **длина строки включает нулевой байт**. В этом отношении `zend_hash` API  отличается от других Zend API, которые не включают нулевой байт в длину строки.

Что это значит на практике? Когда вы передаете в функцию литерал, то длиной строки будет `sizeof("foo")`, а нет `sizeof("foo")-1`. Когда передается строка из zval, длиной строки будет `Z_STRVAL_P(zv)+1`, а не `Z_STRVAL_P(zv)`.

В остальном эти функции работают также как и функции с числовыми индексами:
```c
HashTable *myht;
zval *zv;
zval **zv_dest;

ALLOC_HASHTABLE(myht);
zend_hash_init(myht, 0, NULL, ZVAL_PTR_DTOR, 0);

MAKE_STD_ZVAL(zv);
ZVAL_STRING(zv, "bar", 1);

/* In PHP: $array["foo"] = "bar" */
zend_hash_update(myht, "foo", sizeof("foo"), &zv, sizeof(zval *), NULL);

if (zend_hash_exists(myht, "foo", sizeof("foo"))) {
    php_printf("Key \"foo\" exists\n");
}

if (zend_hash_find(myht, "foo", sizeof("foo"), (void **) &zv_dest) == SUCCESS) {
    php_printf("Fetched value at key \"foo\" into zv_dest\n");
}

if (zend_hash_del(myht, "foo", sizeof("foo")) == SUCCESS) {
    php_printf("Removed value at key \"foo\"\n");
}

if (!zend_hash_exists(myht, "foo", sizeof("foo"))) {
    php_printf("Key \"foo\" no longer exists\n");
}

if (zend_hash_find(myht, "foo", sizeof("foo"), (void **) &zv_dest) == FAILURE) {
    php_printf("As key \"foo\" no longer exists, zend_hash_find returns FAILURE\n");
}

zend_hash_destroy(myht);
FREE_HASHTABLE(myht);
```
Этот код выведет:
```
Key "foo" exists
Fetched value at key "foo" into zv_dest
Removed value at key "foo"
Key "foo" no longer exists
As key "foo" no longer exists, zend_hash_find returns FAILURE
```
Кроме функции `from zend_hash_update()` есть еще одна для вставки значенией со строковыми ключами: `zend_hash_add()`. Разница между этими 2 функциями в том как они себя ведут в случае если переданный ключ уже существует. Функция `zend_hash_update()` перезапишет существующее значение, а `zend_hash_add()` вернет `FAILURE`.

Вот как ведет себя `zend_hash_update()` когда вы пытаетесь перезаписать ключ:
```c
zval *zv1, *zv2;
zval **zv_dest;

/* ... zval init */

zend_hash_update(myht, "foo", sizeof("foo"), &zv1, sizeof(zval *), NULL);
zend_hash_update(myht, "foo", sizeof("foo"), &zv2, sizeof(zval *), NULL);

if (zend_hash_find(myht, "foo", sizeof("foo"), (void **) &zv_dest) == SUCCESS) {
    if (*zv_dest == zv1) {
        php_printf("Key \"foo\" contains zv1\n");
    }
    if (*zv_dest == zv2) {
        php_printf("Key \"foo\" contains zv2\n");
    }
}
```
Код выше напечатает `Key "foo" contains zv2`, то есть значение было перезаписано. Теперь сравним с `zend_hash_add()`:
```c
zval *zv1, *zv2;
zval **zv_dest;

/* ... zval init */

if (zend_hash_add(myht, "bar", sizeof("bar"), &zv1, sizeof(zval *), NULL) == FAILURE) {
    zval_ptr_dtor(&zv1);
} else {
    php_printf("zend_hash_add returned SUCCESS as key \"bar\" was unused\n");
}

if (zend_hash_add(myht, "bar", sizeof("bar"), &zv2, sizeof(zval *), NULL) == FAILURE) {
    zval_ptr_dtor(&zv2);
    php_printf("zend_hash_add returned FAILURE as key \"bar\" is already taken\n");
}

if (zend_hash_find(myht, "bar", sizeof("bar"), (void **) &zv_dest) == SUCCESS) {
    if (*zv_dest == zv1) {
        php_printf("Key \"bar\" contains zv1\n");
    }
    if (*zv_dest == zv2) {
        php_printf("Key \"bar\" contains zv2\n");
    }
}
```
Результатом его работы будет такой вывод:
```
zend_hash_add returned SUCCESS as key "bar" was unused
zend_hash_add returned FAILURE as key "bar" is already taken
Key "bar" contains zv1
```
Здесь второй вызов `zend_hash_add()` возвращает `FAILURE` и в хеш-тиблице остается значение `zv1`.

Помните, что для целочиселнных индексов нет эквивалента функции`zend_hash_add()`. Если вам нужно реализовать подобное поведение, вы или сначала должны сделать проверку существования ключа, или использовать низкоуровневый API:
```c
_zend_hash_index_update_or_next_insert(
    myht, 42, &zv, sizeof(zval *), NULL, HASH_ADD ZEND_FILE_LINE_CC
)
```
Для всех функций перечисленных выше существует второй `quick` (быстрый) вариант, принимающий четвертым аргументом заранее рассчитанное значение хеша. Это позволяет рассчитать хеш единожды и использовать его многократно:
```c
ulong h; /* hash value */

/* ... zval init */

h = zend_get_hash_value("foo", sizeof("foo"));

zend_hash_quick_update(myht, "foo", sizeof("foo"), h, &zv, sizeof(zval *), NULL);

if (zend_hash_quick_find(myht, "foo", sizeof("foo"), h, (void **) &zv_dest) == SUCCESS) {
    php_printf("Fetched value at key \"foo\" into zv_dest\n");
}

if (zend_hash_quick_del(myht, "foo", sizeof("foo"), h) == SUCCESS) {
    php_printf("Removed value at key \"foo\"\n");
}
```
Использование "быстрых" вариантов функций позволяет улучшить производительность так как пропадает необходимость пересчитывать значение хеша на каждый вызов. Разумеется, это имеет смысл только если вы многократно обращаетесь к ключу (напирмер в цикле). "Быстрые" функции в основном используются движком, так как заранее рассчитанные хеши доступны через различные кеши и оптимизации.

##Применение функций
Часто вам нужно работать не с конкретным ключом, а нужно выполнить операцию над всеми значениями хеш-таблицы. PHP для этого предлагает два механизма: первый — это семейство функций `zend_hash_apply_*()` которые вызывают переданную функцию для всех элементов хеш-таблицы. Всего доступно 3 таких функции:
```c
void zend_hash_apply(HashTable *ht, apply_func_t apply_func TSRMLS_DC);
void zend_hash_apply_with_argument(
    HashTable *ht, apply_func_arg_t apply_func, void *argument TSRMLS_DC
);
void zend_hash_apply_with_arguments(
    HashTable *ht TSRMLS_DC, apply_func_args_t apply_func, int num_args, ...
);
```
Все три функции делают одно и то же, но принимают разное число аргументов к функции `apply_func`. Ниже приведены соответствующие сигнатуры для `apply_func`-ций:
```c
typedef int (*apply_func_t)(void *pDest TSRMLS_DC);
typedef int (*apply_func_arg_t)(void *pDest, void *argument TSRMLS_DC);
typedef int (*apply_func_args_t)(
    void *pDest TSRMLS_DC, int num_args, va_list args, zend_hash_key *hash_key
);
```
Как вы видите `zend_hash_apply()` не передает дополнительных аргументов в коллбэк, `zend_hash_apply_argument()` передает один дополнительный аргумент и `zend_hash_apply_with_arguments()` передает произвольное число аргументов (за это отвечает `va_list args`). Кроме того, последняя фнукция передаетне только значение `void *pDest`, но и соответствующий `hash_key`. Структура `zend_hash_key` выглядит так:
```c
typedef struct _zend_hash_key {
    const char *arKey;
    uint nKeyLength;
    ulong h;
} zend_hash_key;
```
Члены-данных здесь имеют тот же смысл что и в `Bucket`. Если `nKeyLength == 0`, значит `h` — это целочисленный ключ. Иначе это хеш строкового ключа `arKey` длиной `nKeyLength`.

Для примера давайте разработаем дампер массивов похожий на `var_dump`. Мы будем использовать `zend_hash_apply_with_arguments()`, но не потому что мы должны передавать много аргументов, а потому что нам нужен ключ массива. Начнем с основной функции:
```c
static void dump_value(zval *zv, int depth) {
    if (Z_TYPE_P(zv) == IS_ARRAY) {
        php_printf("%*carray(%d) {\n", depth * 2, ' ', zend_hash_num_elements(Z_ARRVAL_P(zv)));
        zend_hash_apply_with_arguments(Z_ARRVAL_P(zv), dump_array_values, 1, depth + 1);
        php_printf("%*c}\n", depth * 2, ' ');
    } else {
        php_printf("%*c%Z\n", depth * 2, ' ', zv);
    }
}

PHP_FUNCTION(dump_array) {
    zval *array;

    if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "a", &array) == FAILURE) {
        return;
    }

    dump_value(array, 0);
}
```
Код выше использует некоторые опции функции `php_printf()`, которые нужно упомянуть. `%*c` повторяет символ несколько раз. Так `php_printf("%*c", depth * 2, ' ')` выводит пробел `depth * 2` раз, этот код отвечает за увеличение отступов при увеличении глубины массива. `%Z` конвертирует zval в строку и печатает его.

Таким образом, код выше печатает значения используя `%Z`, но массивы он обрабатывает особым образом: он добавляет обертку `array(n) { ... }`. Вот так задается функция, применяемая ко всем элементам хеш-таблицы:
```c
zend_hash_apply_with_arguments(Z_ARRVAL_P(zv), dump_array_values, 1, depth + 1);
```
`dump_array_values` это коллбэк, который будет вызван для каждого элемента. `1` — число передаваемых аргументов и `depth + 1` этот (единственный) аргумент. Вот как может выглядеть эта функция:
```c
static int dump_array_values(
    void *pDest TSRMLS_DC, int num_args, va_list args, zend_hash_key *hash_key
) {
    zval **zv = (zval **) pDest;
    int depth = va_arg(args, int);

    if (hash_key->nKeyLength == 0) {
        php_printf("%*c[%ld]=>\n", depth * 2, ' ', hash_key->h);
    } else {
        php_printf("%*c[\"", depth * 2, ' ');
        PHPWRITE(hash_key->arKey, hash_key->nKeyLength - 1);
        php_printf("\"]=>\n");
    }

    dump_value(*zv, depth);

    return ZEND_HASH_APPLY_KEEP;
}
```
Переданный аргумент `depth` получен так: `depth = va_arg(args, int)`. Все остальные аргументы могут быть извлечены таким же способом. Дальше идет код для красивого форматирования ключей и рекурсивный вызов `dump_value` для вывода значения.

В самом конце функция возвращает `ZEND_HASH_APPLY_KEEP`, что является одним из 4 разрешенных значений для такого типа коллбэков, ниже приведены разрешенные коды возврата:

`ZEND_HASH_APPLY_KEEP`:
    Сохранить посещенный элемент и продолжить обход хеш-таблицы.
`ZEND_HASH_APPLY_REMOVE`:
    Удалить посещенный элемент и продолжить обход хеш-таблицы.
`ZEND_HASH_APPLY_STOP`
    Сохранить посещенный элемент и остановить обход хеш-таблицы.
`ZEND_HASH_APPLY_REMOVE | ZEND_HASH_APPLY_STOP`
    Удалить посещенный элемент и остановить обход хеш-таблицы.
Таким образом, функции `zend_hash_apply_*()` могут вести себя и как `array_map()`, и как `array_filter()`, а также имеют дополнительную возможность прервать итерирование.

Давайте испытаем нашу фнукцию:
```c
dump_array([1, [2, "foo" => 3]]);
// output:
array(2) {
  [0]=>
  1
  [1]=>
  array(2) {
    [0]=>
    2
    ["foo"]=>
    3
  }
}
```
Этот результат выглядит похожим на результат работы `var_dump`. Если вы взглянете на код функции `php_var_dump()`, вы поймете, что там реализован аналогичный подход.

##Итерирование
Второй способ выполнить операцию над всеми значениями хэщ-таблицы это итерация через неё. Итерирование хеш-таблицы в C очень похоже на итерирование вручную в PHP:
```
<?php

for (reset($array);
     null !== $data = current($array);
     next($array)
) {
    // Do something with $data
}
```
Эквивалентный C-код для обзора массива выглядит так:
```c
zval **data;

for (zend_hash_internal_pointer_reset(myht);
     zend_hash_get_current_data(myht, (void **) &data) == SUCCESS;
     zend_hash_move_forward(myht)
) {
    /* Do something with data */
}
```
Сниппет выше использует внутренний указатель массива (`pInternalPointer`), обычно это плохая идея использовать его. Этот указатель является частью хеш-таблицы и он совместно используется всеми частями кода взаимодействующими с массивом. Например, вложенная итерация по хеш-таблице невозможна при использовании внутреннего указателя массива (так как один цикл изменит указатель другого).

По этой причине все итерирующие функции используют второй вариант функций с суффикосм `_ex`, которые работают с внешним указателем позиции. При использовании этого API текущая позиция хранится в `HashPosition` (это просто typedef для `Bucket *`) и указатель на эту структуру передается последним аргументом во все функции:
```c
HashPosition pos;
zval **data;

for (zend_hash_internal_pointer_reset_ex(myht, &pos);
     zend_hash_get_current_data_ex(myht, (void **) &data, &pos) == SUCCESS;
     zend_hash_move_forward_ex(myht, &pos)
) {
    /* Do something with data */
}
```
Также можно обходить хеш-таблицу в обратном направлении если использовать `end` вместо `reset` и `move_backwards` вместо `move_forward`:
```c
HashPosition pos;
zval **data;

for (zend_hash_internal_pointer_end_ex(myht, &pos);
     zend_hash_get_current_data_ex(myht, (void **) &data, &pos) == SUCCESS;
     zend_hash_move_backwards_ex(myht, &pos)
) {
    /* Do something with data */
}
```
В дополнение вы можете извлечь ключ используя функцию `zend_hash_get_current_key_ex()`, у неё такая сигнатура:
```c
int zend_hash_get_current_key_ex(
    const HashTable *ht, char **str_index, uint *str_length,
    ulong *num_index, zend_bool duplicate, HashPosition *pos
);
```
Эта функция возвращает тип ключа, который может принимать следующие значения:

`HASH_KEY_IS_LONG`:
    Ключ это целое число, которое может быть записано в `num_index`.
`HASH_KEY_IS_STRING`:
    Ключ это строка, которая может быть записана в `str_index`. Параметр `duplicate` определяет должен ли ключ записан напрямую или его нужно скопировать. Длина строки (еще раз подчеркиваем, что в этом контексте она включает нулевой байт) записывается в `str_length`.
`HASH_KEY_NON_EXISTANT`:
    Это значит, что мы при итерации достигли конца хеш-таблицы и в ней больше нет элементов. В циклах описанных выше этот случай невозможен.

Для разбора различных возвращаемых значений эта функция обычно используется вннутри оператора `switch`:
```с
char *str_index;
uint str_length;
ulong num_index;

switch (zend_hash_get_current_key_ex(myht, &str_index, &str_length, &num_index, 0, &pos)) {
    case HASH_KEY_IS_LONG:
        php_printf("%ld", num_index);
        break;
    case HASH_KEY_IS_STRING:
        /* Subtracting 1 as the hashtable lengths include the NUL byte */
        PHPWRITE(str_index, str_length - 1);
        break;
}
```
В PHP 5.5 есть дополнительная функция `zend_hash_get_current_key_zval_ex()`, которая упрощает запись ключа в zval:
```c
zval *key;
MAKE_STD_ZVAL(key);
zend_hash_get_current_key_zval_ex(myht, key, &pos);
```
##Копирование и объединение

Копирование хеш-таблицы это еще одна частовстречающаяся операция. Обычно вы не должны длеать это вручную, так как PHP сделает это сам при возникновении копирования-при-записи. Копирование выполняется при помощи функции `zend_hash_copy()`:
```c
HashTable *ht_source = get_ht_from_somewhere();
HashTable *ht_target;

ALLOC_HASHTABLE(ht_target);
zend_hash_init(ht_target, zend_hash_num_elements(ht_source), NULL, ZVAL_PTR_DTOR, 0);
zend_hash_copy(ht_target, ht_source, (copy_ctor_func_t) zval_add_ref, NULL, sizeof(zval *));
```
Четвертый аргумент функции `zend_hash_copy()` больше не используется, он всегда должен иметь значение `NULL`. Третий аргумент — это *копирующий конструктор*, который будет вызван для каждого копируемого элемента. Для zval-ов это будет функция `zval_add_ref`, которая просто добавляет дополнительную ссылку ко всем элементам.

Функция `zend_hash_copy()` работает и в случае если целевая хеш-таблица уже содержит элементы. Если ключ из `ht_source` уже существует в `ht_target`, то он будет перезаписан. Чтобы контролировать такое поведение может быть использована функция объединения  `zend_hash_merge()`, она имеет ту же сигнатуру что и `zend_hash_copy()`, но принимает дополнительный аргумент, который определяет должна или нет осуществляться перезапись.

`zend_hash_merge(..., 0)` будет осуществлять копирование только эсли элемент с таким ключом не существует в целевой хеш-таблице. `zend_hash_merge(..., 1)` ведет себя почти также как и `zend_hash_copy()`. Отличие только в том, что `merge` устанавливает внутренний указатель на первый эелмент (`pListHead`), а `copy` устанавливает его в то же значение что и в хеш-таблице источнике.

Для более полного контроля поведения при объединении хеш-таблиц может использоваться функция `zend_hash_merge_ex`, которая определяет какие элементы должны быть скопированы. Для этого используется специальная функция проверки (merge checker function):
```c
typedef zend_bool (*merge_checker_func_t)(
    HashTable *target_ht, void *source_data, zend_hash_key *hash_key, void *pParam
);
```
Функция проверки принимает целевую хеш-тиблицу, исходные данные, хеш ключа и дополнительный аргумент (такой же как в `zend_hash_apply_with_argument()`). Для примера давайте реализуем функцию, которая берет два массива, объединяет их и в случае коллизии ключей использует б*о*льшее значение:
```c
static int merge_greater(
    HashTable *target_ht, zval **source_zv, zend_hash_key *hash_key, void *dummy
) {
    zval **target_zv;
    zval compare_result;

    if (zend_hash_quick_find(
            target_ht, hash_key->arKey, hash_key->nKeyLength, hash_key->h, (void **) &target_zv
        ) == FAILURE
    ) {
        /* Key does not exist in target hashtable, so copy in any case */
        return 1;
    }

    /* Copy only if the source zval is greater (compare == 1) than the target zval */
    compare_function(&compare_result, *source_zv, *target_zv);
    return Z_LVAL(compare_result) == 1;
}

PHP_FUNCTION(array_merge_greater) {
    zval *array1, *array2;

    if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "aa", &array1, &array2) == FAILURE) {
        return;
    }

    /* Copy array1 into return_value */
    RETVAL_ZVAL(array1, 1, 0);

    zend_hash_merge_ex(
        Z_ARRVAL_P(return_value), Z_ARRVAL_P(array2), (copy_ctor_func_t) zval_add_ref,
        sizeof(zval *), (merge_checker_func_t) merge_greater, NULL
    );
}
```
В основной функции сначала массив `array1` копируется в возвращаемое значение, затем объединяется с `array2`. Здесь `array1` это целевой массив, к которому будут добавлены элементы `array2`. При объединении проверочная функция `merge_greater()` вызывается для всех элементов второго массива. Эта функция сначала пытается найти элемент с проверяемым ключом в первом массиве. Если такого элемента нет, то он будет скопирован из второго массива в первый. Если элемент есть, то он будет скопирован если значение во втором массиве больше чем в первом.

Давайте проверим эту функцию:
```c
var_dump(array_merge_greater(
    [3 => 0, "bar" => -5],
    ["bar" => 5, "foo" => -10, 3 => -42]
));
// output:
array(3) {
  [3]=>
  int(0)
  ["bar"]=>
  int(5)
  ["foo"]=>
  int(-10)
}
```
##Сравнение, сортировка и экстремумы

Последние три функции в API хеш-таблиц связаны с разного рода сравнением элементов хеш-таблицы друг с другом. Сравнение определено *функцией сравнения*:
```c
typedef int (*compare_func_t)(const void *left, const void *right TSRMLS_DC);
```
Эта функция принимает два элемента хеш-таблиц и возрвщает то, как они друг с другом соотносятся. Отрицательное число возвращается если `left < right`, положительное означает что `left > right`, 0 — элементы равны.

Первая функция которую мы рассмотрим это `zend_hash_compare()`, она сравнивает 2 хеш-таблицы:
```c
int zend_hash_compare(
    HashTable *ht1, HashTable *ht2, compare_func_t compar, zend_bool ordered TSRMLS_DC
);
```
Возвращаемое значение имеет тот же смысл что и в описанной выше функции `compare_func_t`. Функция сначала сравнивает размеры массивов, если они отличаются, то массив с большим размером считается больше. То что происходит если массивы имеют одинаковый размер зависит от параметра `ordered`:

Если `ordered=0` (порядок элементов не учитывается) функция пройдется по всем бакетам первой хеш-таблицы и попробует найти элемент с таким же ключом во второй. Если какого-то ключа нет, то первый массив будет считаться больше второго. Если все ключи есть, то функция сравнит значения.

Если `ordered=1` (порядок элементов учитывается), то обе хеш-таблицы будут обходиться одновременно. Для каждого элемента сначала сравниваются ключи и если они совпадают, то сравниваются значения c использованием `compar`.

Это продолжается до тех пор пока одна из операций сравнения вренет ненулевое значение (в таком случае результат сравнения  также будет результатом `zend_hash_compare()`) или пока не останется элементов для сравнения. В последнем случае хеш-таблицы будут считаться эквивалентными.

Оба режима сравнения могут быть связаны с поведением в PHP операторов сравнения:
```c
/* $ar1 == $ar2 compares the elements with == and does not take order into account: */
zend_hash_compare(ht1, ht2, (compare_func_t) hash_zval_compare_function, 0 TSRMLS_CC);

/* $ar1 === $ar2 compares the elements with === and takes order into account: */
zend_hash_compare(ht1, ht2, (compare_func_t) hash_zval_identical_function, 1 TSRMLS_CC);
```

Следующая функция, которую мы рассмотрим, `zend_hash_sort()`, она используется для сортировки хеш-таблицы:
```c
int zend_hash_sort(HashTable *ht, sort_func_t sort_func, compare_func_t compar, int renumber TSRMLS_DC);
```
Эта функция выполняет только пред- и постобработку хеш-таблицы, а саму сортировку делегирует функции переданной в `sort_func`:
```с
typedef void (*sort_func_t)(
    void *buckets, size_t num_of_buckets, register size_t size_of_bucket,
    compare_func_t compare_func TSRMLS_DC
);
```
Функция получит массив бакетов, их число и размер (это всегда `sizeof(Bucket *)`), а также функцию сравнения. "Массив бакетов" здесь это обычный массив C, а не хеш-таблица. Функция сортировки пройдется по всем бакетам в этом массиве и переопределит их порядок.

После того как функция сортировки завершит работу `zend_hash_sort()` переконструирует хеш-тиблицу на основе массива C. Если параметр `renumber=0`, то значения сохранят свои ключи и только поменяют порядок. Если `renumber=1`, то массив будет перенумерован и результирующая хеш-таблица будет иметь возрастающие челочисленные ключи.

Пока вы не захотите реализовать собственный алгоритм сортировки будет использоваться функция `zend_qsort`, это PHP реализация алгоритма quicksort.

Последняя среди функций связанных со сравниванием это функция поиска максимального и минимального элементов хеш-таблицы:
```c
int zend_hash_minmax(
    const HashTable *ht, compare_func_t compar, int flag, void **pData TSRMLS_DC
);
```
Если задан `flag=0`, то минимальное значение будетзаписано в `pData`, если задано `flag=1` — максимальное. Если хеш-таблица пуста, то функция вернет `FAILURE` (так как невозможно определить min/max для неправильно определенного или пустого массива).
