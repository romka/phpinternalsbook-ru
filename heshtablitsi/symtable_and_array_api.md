# Symtable и API массивов

API хеш-таблиц позволяет вам работать со значениями любых типов, но в большинстве случаев вам предстоит работать с zval-ами. Использование `zend_hash` API с zval-ами может быть чем-то избыточным, так как вам приходится задумываться о выделении памяти и инициализации. По этой причине PHP предоставляет второй набор API. Перед тем как разбираться с этим упрощенным API мы взглянем на некоторые особенности хеш-таблиц, которые использует PHP.

## Symtables

Одна из концепций дизайна ядра PHP состоит в том, что целые числа и строки содержащие целые числа должны быть взаимозаменяемы. Это, в том числе, означает что в массивах кдючи `42` и `"42"` должны считаться одинаковыми. Это не так в случае обычных хеш-таблиц, они строго различают типы ключей и они могут содержать одновременно ключи `42` и `"42"` с разными значениями.

По этой причине реализован дополнительный *symtable* (символьная таблица) API, который является тонкой оберткой вокруг некоторых функций для работы с хеш-таблицами. Этот API конвертирует строковые ключи к актуальным целочисленным значениям. Например вот так определена функция `zend_symtable_find()`:
```c
static inline int zend_symtable_find(
    HashTable *ht, const char *arKey, uint nKeyLength, void **pData
) {
    ZEND_HANDLE_NUMERIC(arKey, nKeyLength, zend_hash_index_find(ht, idx, pData));
    return zend_hash_find(ht, arKey, nKeyLength, pData);
}
```
Реализация макроса `ZEND_HANDLE_NUMERIC()` не будет рассмотрена здесь, важна только его функциональность. Если `arKey` содержит целое число между `LONG_MIN` и `LONG_MAX`, то это число будет записано в `idx` и функция `zend_hash_index_find()` будет вызвана для него. В остальных случаях код перейдет к следующей строке где вызывается `zend_hash_find()`.

Кроме `zend_symtable_find()` следующие функции также являются частью symtable API,  они имеют то же поведение, что и их аналоги работающие с хеш-таблицами, но они также включают нормализацию строковых ключей к целочисленным:
```c
static inline int zend_symtable_exists(HashTable *ht, const char *arKey, uint nKeyLength);
static inline int zend_symtable_del(HashTable *ht, const char *arKey, uint nKeyLength);
static inline int zend_symtable_update(
    HashTable *ht, const char *arKey, uint nKeyLength, void *pData, uint nDataSize, void **pDest
);
static inline int zend_symtable_update_current_key_ex(
    HashTable *ht, const char *arKey, uint nKeyLength, int mode, HashPosition *pos
);
```
Кроме того, есть еще два макроса для создания сим-таблиц:
```c
#define ZEND_INIT_SYMTABLE_EX(ht, n, persistent) \
    zend_hash_init(ht, n, NULL, ZVAL_PTR_DTOR, persistent)

#define ZEND_INIT_SYMTABLE(ht) \
    ZEND_INIT_SYMTABLE_EX(ht, 2, 0)
```
Как вы видите, эти макросы просто вызывают `zend_hash_init()` используя `ZVAL_PTR_DTOR` в качестве деструктора. Эти макросы не связаны напрямую с преобразованием строк к числам описанным выше.

Давайте испытаем эти функции:
```c
HashTable *myht;
zval *zv1, *zv2;
zval **zv_dest;

ALLOC_HASHTABLE(myht);
ZEND_INIT_SYMTABLE(myht);

MAKE_STD_ZVAL(zv1);
ZVAL_STRING(zv1, "zv1", 1);

MAKE_STD_ZVAL(zv2);
ZVAL_STRING(zv2, "zv2", 1);

zend_hash_index_update(myht, 42, &zv1, sizeof(zval *), NULL);
zend_symtable_update(myht, "42", sizeof("42"), &zv2, sizeof(zval *), NULL);

if (zend_hash_index_find(myht, 42, (void **) &zv_dest) == SUCCESS) {
    php_printf("Value at key 42 is %Z\n", *zv_dest);
}

if (zend_symtable_find(myht, "42", sizeof("42"), (void **) &zv_dest) == SUCCESS) {
    php_printf("Value at key \"42\" is %Z\n", *zv_dest);
}

zend_hash_destroy(myht);
FREE_HASHTABLE(myht);
```
Этот код выведет:
```c
Value at key 42 is zv2
Value at key "42" is zv2
```
Таким образом, оба вызова `update` запишут данные в один элемент (второй перезапишет первый) и оба вызова `find` найдут один и тот же элемент.

##API массивов

Теперь у нас есть всё необходимое для изучения API массивов. Этот API работает не напрямую с хеш-таблицами, а принимает zval-ы, из которых извлекаются хеш-таблицы при помощи `Z_ARRVAL_P()`.

Первые две функции этого API это `array_init()` и `array_init_size()`, они инициализируют хеш-таблицу в zval-е. Первая функция принимает только целевой zval, а вторая также принимает size hint:
```c
/* Create empty array into return_value */
array_init(return_value);

/* Create empty array with expected size 1000000 into return_value */
array_init_size(return_value, 1000000);
```
Оставшиеся функции из этого API все связаны со вставкой значений в массив. Есть 4 семейства функций:

```c
/* Insert at next index */
int add_next_index_*(zval *arg, ...);
/* Insert at specific index */
int add_index_*(zval *arg, ulong idx, ...);
/* Insert at specific key */
int add_assoc_*(zval *arg, const char *key, ...);
/* Insert at specific key of length key_len (for binary safety) */
int add_assoc_*_ex(zval *arg, const char *key, uint key_len, ...);
Here * is a placeholder for a type and ... a placeholder for the type-specific arguments. The valid values for them are listed in the following table:
```

|Type|  Additional arguments|
|----|-----------------------|
|`null`|  none|
|`bool`|  `int b`|
|`long`|  `long n`|
|`double`|  `double d`|
|`string`|  `const char *str, int duplicate`|
|`stringl`| `const char *str, uint length, int duplicate`|
|`resource`|  `int r`|
|`zval`|  `zval *value`|
Для того чтобы поэкспериментировать с этими функциями давайте создадим тестовый массив с элементами разных типов:
```c
PHP_FUNCTION(make_array) {
    zval *zv;

    array_init(return_value);

    add_index_long(return_value, 10, 100);
    add_index_double(return_value, 20, 3.141);
    add_index_string(return_value, 30, "foo", 1);

    add_next_index_bool(return_value, 1);
    add_next_index_stringl(return_value, "\0bar", sizeof("\0bar")-1, 1);

    add_assoc_null(return_value, "foo");
    add_assoc_long(return_value, "bar", 42);

    add_assoc_double_ex(return_value, "\0bar", sizeof("\0bar"), 1.61);

    /* For some things you still have to manually create a zval... */
    MAKE_STD_ZVAL(zv);
    object_init(zv);
    add_next_index_zval(return_value, zv);
}
```
`var_dump()` выведет следующее (NUL-байты заменены на `\0`):
```
array(9) {
  [10]=>
  int(100)
  [20]=>
  float(3.141)
  [30]=>
  string(3) "foo"
  [31]=>
  bool(true)
  [32]=>
  string(4) "\0bar"
  ["foo"]=>
  NULL
  ["bar"]=>
  int(42)
  ["\0bar"]=>
  float(1.61)
  [33]=>
  object(stdClass)#1 (0) {
  }
}
```
Взглянув на API массивов вы можете обратить внимание, что API массивов еще более неоднороден в отношении длин строк. Длины передаваемые в `_ex` функции *включают* завершающий нулевой байт, а длины передаваемые в `stringl` функции *не включают*.

Также вам следует помнить, что функции начинающиеся с `add` ведут себя как `update` функции, то есть перезаписывают существующие ключи.

Также существует несколько `add_get` функций, которые и вставляют, и извлекают значение (по аналогии с последним параметром функции `zend_hash_update`). Эти функции почти никогда не используются и не будут обсуждаться здесь. Они упомянуты просто для полноты картины.

Здесь завершается наш обзор хеш-таблиц, сим-таблиц и API массивов.
