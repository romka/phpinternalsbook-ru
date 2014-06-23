# API хештаблиц

В PHP есть два набора API для работы с хэш-таблицами:  первый набор это низкоуровневый `zend_hash` API, он рассмотрен в этом разделе. Второй набор — это API для работы с массивами, который предоставляет высокоуровненвые функции для выполнения распространенных операций. Второй набор рассмотрен в следующем разделе.

##Создание и удаление хэш-таблиц

Память для хэш-таблиц выделяется с использованием `ALLOC_HASHTABLE()`, инициализация выполняется при помощи `zend_hash_init()`:
```c
HashTable *myht;

/* Same as myht = emalloc(sizeof(HashTable)); */
ALLOC_HASHTABLE(myht);

zend_hash_init(myht, 1000000, NULL, NULL, 0);
```
Второй аргумент в `zend_hash_init()` это size hint (подсказка по размеру хэш-таблицы), который определяет как много элементов вы ожидаете иметь в хэш-таблице. В случае когда передано значение 1000000 PHP выделит место под for 2^20 = 1048576 элементов в момент первой вставки. Без такой подсказки PHP сначала выделит место под 8 элементов и затем будет выполнять множество изменений размера хэш-таблицы при вставке новых элементов (сначала до 16, потом до 32, до 64 и т.д). Каждое изменение размера требует чтобы память под `arBuckets` была выделена заново (reallocated), а также требует пересчета всех хэшей (“rehash”), что приводит к пересчету коллизий.

Использование  size hint помогает избежать ненужных операций изменения размера и улучшает производительность. Но это имеет смысл только для больших хэш-таблиц, для маленьких достаточно установить size hint в 0. Так как обычно минимальный размер таблицы равен 8, то нет разницы каким вы сделаете сайз хинт 0, 2, или 7.

Третий аргумент функции `zend_hash_init()` всегда должен иметь значение `NULL`. Ранее он использовался для указания кастомной хэш-функции, но сейчас этот аргумент больше не поддерживается. Четвертый аргумент — это функция-деструктор, применяемая для хрантимых значений, она имеет такую сигнатуру:
```c
typedef void (*dtor_func_t)(void *pDest);
```
В большинстве случаев используется ` ZVAL_PTR_DTOR` (для хранимых значений типа `zval *`). Это обычный `zval_ptr_dtor()`, но его сигнатура совместима с `dtor_func_t`.

Последний аргумент функции `zend_hash_init()` определяет следует ли использовать persistent allocation. Если вы хотите, чтобы хэш-таблица существовала после завершения запроса этот аргумент должен быть равен `1`. Также существует другая функция инициализации `zend_hash_init_ex()`, она принимает дополнительный аргумент `bApplyProtection` типа boolean. Установка его в 0 позволяет отключить защиту от рекурсии (которая включена по умолчанию). Эта функция используется редко, обычно для внутрениих структур движка (таких как функции или class table).

Хэш-таблица может быть уничтожена с помощью `zend_hash_destroy()` и освобождена с помощью `FREE_HASHTABLE()`:
```c
zend_hash_destroy(myht);

/* Same as efree(myht); */
FREE_HASHTABLE(myht);
```
Функция `zend_hash_destroy()` вызовет деструктор для всех бакетов и освободит их. Пока эта функция выполняется хэш-таблица находится в несогласованном  (inconsistent) состоянии и не может быть использована. Обычно это не проблема, но в некоторых редких случаях (особенно когда деструктор может быть вызван в пользовательском коде) может понадобиться, чтобы хэш-таблица могла быть использована в процессе работы деструктора. В этом случае могут быть использованы функции `zend_hash_graceful_destroy()` и `zend_hash_graceful_reverse_destroy()`. Первая функция удаляет бакеты в порядке, в котором они были вставлены, вторая — в обратном порядке.

Если вам нужно удалить все элементы хэш-таблицы, но не уничтожить саму хэш-таблицу, вы можете использовать функцию `zend_hash_clean()`.

##Числовые ключи

Прежде чем изучать функции использующиеся для вставки, извлечения и удаления числовых ключей в хэш-таблицы, давайте проясним какие аргументы нужны этим функциям.

Вспомним, что член-данных бакета `pData` хранит указатель на реальные данные. Таким образом, если в хэш-таблице вы храните указатели `zval *`, тогда `pData` будет `zval **`. По этой причине вставка в хэш-таблицу потребует передачи `zval **`, несмотря на то, что вы определили тип данных как `zval *`.

Когда вы извлекаете значения из хэш-таблицы вам нужно будет передать в функцию указатель на переменную назначения `pDest`, в которую будет записано значение `pData`. Для того чтобы записать в указатель используя `*pDest = pData` нужен еще один indirection уровень. Итак, если вы записываете в хэш-таблицу `zval *`, то в функцию для извлечения данных вы должны будете передать `zval ***`.

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
Пример выше вставляет в хэш-таблицу `zval *` содержащий `"foo"` с ключом `42`. The fourth argument specifies the used data type: `sizeof(zval *)`. Третий аргумент, вставляемое значение, должен иметь тип `zval **`.

Последний аргумент может быть использован и для вставки значения, и для извлечения:
```c
zval **zv_dest

zend_hash_index_update(myht, 42, &zv, sizeof(zval *), (void **) &zv_dest);
```
Зачем это может понадобиться? Ведь вы и так знаете значение, которое вы вставили в хэш-таблицу, зачем его извлекать? Помните, что хэш-таблицы работают с *копией* переданного значения. Таким образом, хотя `zval *` сохраненный в хэш-таблице имеет то же значение что и `zv`, но он хранится по другому адресу. Если вы захотите модифицировать это новое значение из хэш-таблицы, то вам нужен указатель на это значение, и это как раз то, что будет записано в `zv_dest`.

При сохранении значений `zval *` последний аргумент функции `zend_hash_index_update` обычно не используется. Однако, при использовании значений, а не указателей, вы часто будете встречаться с паттерном, когда сначала создается временная структура, которая затем вставляется в хэш-таблицу и значение указателя назначения (`**zv_dest`) используется для дальнейшей работы (так как изменения во временной структуре не будут затрагивать значение в хэш-таблице).

Часто вам не нужно вставлять значение с конкретным индексом, достаточно добавить его в конец хэш-таблицы. Это можно сделать с помощью функции `zend_hash_next_index_insert()`:
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
Что здесь происходит? Первое значение вставляется с ключом `LONG_MAX`. В этом случае следующим целочисленным ключом должен быть `LONG_MAX + 1`, который выходит за пределы `LONG_MIN`. Такое переполнение недопустимо, PHP делает проверку и, при необходимости, оставляет `nNextFreeElement` равным `LONG_MAX`. Когда `zend_hash_next_index_insert()` запустится, то он попробует вставить значение с ключом `LONG_MAX`, но этот ключ уже занят, по этому функция вернет ошибку..

Последний пример кода использует две новых для нас функции. Одна из них возвращает следующий свободный целочисленный ключ (который, как мы убедились, не обязательно будет свободным), вторая возвращает число элемкнтов в хэш-таблице. Функция `zend_hash_num_elements()` используется довольно часто.

Назначение следующих трех функций должно быть очевидно: `zend_hash_index_find()` возвращает индекс по хэшу, `zend_hash_index_exists()` проверяет существует ли индекс без извлечения значения и `zend_hash_index_del()` удаляет элемент. Пример использования функций:
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

Работа со строковыми ключами очень похожа на работу с целочисленными. Основное отличие состоит в том, что ключевое слово `index` отсутствует в именах функций. Эти функции принимают на вход строковый индекс и его длину, а не числовой индекс.

Важно понимать, что а данном контексте означает "длина строки": в API хэш-таблиц **длина строки включает нулевой байт**. В этом отношении `zend_hash` API  отличается от других Zend API, которые не включают нулевой байт в длину строки.

Что это значит на практике? Когда вы передаете в функцию литерал, то длиной строки будет `sizeof("foo")`, а нет `sizeof("foo")-1`. Когда передается строка из zval, длиной строки будет `Z_STRVAL_P(zv)+1`, а нет `Z_STRVAL_P(zv)`.

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
Здесь второй вызов `zend_hash_add()` возвращает `FAILURE` и в хэш-тиблице остается значение `zv1`.

Помните, что для целочиселнных индексов нет эквивалента функции`zend_hash_add()`. Если вам нужно реализовать подобное поведение, вы или сначала должны сделать проверку существования ключа, или использовать низкоуровневый API:
```c
_zend_hash_index_update_or_next_insert(
    myht, 42, &zv, sizeof(zval *), NULL, HASH_ADD ZEND_FILE_LINE_CC
)
```
Для всех функций перечисленных выше существует второй `quick` (быстрый) вариант, принимающий четвертым аргументом заранее рассчитанное значение хэша. Это позволяет рассчитать хэш единожды и использовать его многократно:
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
Использование "быстрых" вариантов функций позволяет улучшить производительность так как пропадает необходимость пересчитывать значение хэша на каждый вызов. Разумеется, это имеет смысл только если вы многократно обращаетесь к ключу (напирмер в цикле). "Быстрые" функции в основном используются движком, так как заранее рассчитанные хэши доступны через различные кеши и оптимизации.

##Применение функций
Часто вам нужно работать не с конкретным ключом, а нужно выполнить операцию над всеми значениями хэш-таблицы. PHP для этого предлагает два механизма: первый — это семейство функций `zend_hash_apply_*()` которые вызывают переданную функцию для всех элементов хэш-таблицы. Всего доступно 3 таких функции:
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
Члены-данных здесь имеют тот же смысл что и в `Bucket`. Если `nKeyLength == 0`, значит `h` — это целочисленный ключ. Иначе это хэш строкового ключа `arKey` длиной `nKeyLength`.

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

Таким образом, код выше печатает значения используя `%Z`, но массивы он обрабатывает особым образом: он добавляет обертку `array(n) { ... }`. Вот так задается функция, применяемая ко всем элементам хэш-таблицы:
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
    Сохранить посещенный элемент и продолжитьт обход хэш-таблицы.
`ZEND_HASH_APPLY_REMOVE`:
    Удалить посещенный элемент и продолжить обход хэш-таблицы.
`ZEND_HASH_APPLY_STOP`
    Сохранить посещенный элемент и остановить обход хэш-таблицы.
`ZEND_HASH_APPLY_REMOVE | ZEND_HASH_APPLY_STOP`
    Удалить посещенный элемент и остановить обход хэш-таблицы.
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
