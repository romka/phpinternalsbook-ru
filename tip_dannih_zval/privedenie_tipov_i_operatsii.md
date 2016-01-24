# Приведение типов и операции

##Базовые операции

Так как zval-ы это комплекстные значения вы не можете напрямую выполнять над ними такие операции как `zv1 + zv2`. Если вы попробуете выполнить что-то подобное, то либо получите ошибку, либо получите сложение двух указателей, а не их значений.

"Базовые" операции, такие как `+`, сложны когда вы работаете с zval-ами, так как вам нужно работать с разными типами данных. Например, PHP позволяет складывать переменную типа double и строку, содержащую число (3.14 + "17") или даже сложить два массива ([1, 2, 3] + [4, 5, 6])..

Для этих целей PHP предоставляет специальные функции, выполняющие операции над zval-ами. Например, сложение производится функцией `add_function()`:
```с
zval *a, *b, *result;
MAKE_STD_ZVAL(a);
MAKE_STD_ZVAL(b);
MAKE_STD_ZVAL(result);

ZVAL_DOUBLE(a, 3.14);
ZVAL_STRING(b, "17", 1);

/* result = a + b */
add_function(result, a, b TSRMLS_CC);

php_printf("%Z\n", result); /* 20.14 */

/* zvals a, b, result need to be dtored */
```
Кроме функции `add_function()` есть и другие функции реализующие бинарные операции (с двумя операндами), все они имеют одинаковую сигнатуру:
```c
int add_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);                 /*  +  */
int sub_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);                 /*  -  */
int mul_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);                 /*  *  */
int div_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);                 /*  /  */
int mod_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);                 /*  %  */
int concat_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);              /*  .  */
int bitwise_or_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);          /*  |  */
int bitwise_and_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);         /*  &  */
int bitwise_xor_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);         /*  ^  */
int shift_left_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);          /*  << */
int shift_right_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);         /*  >> */
int boolean_xor_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);         /* xor */
int is_equal_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);            /*  == */
int is_not_equal_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);        /*  != */
int is_identical_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);        /* === */
int is_not_identical_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);    /* !== */
int is_smaller_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);          /*  <  */
int is_smaller_or_equal_function(zval *result, zval *op1, zval *op2 TSRMLS_DC); /*  <= */
```
Все функции принимают в качестве аргумента zval `result`, в которой будет сохранен результат операции над операндами `op1` и `op2`. Результат возвращаемый функциями (`SUCCESS` или `FAILURE`) сообщает о том была операция произведена успешно или нет. Помните, что переменной `result` всегда устанавливается какое-то значения, вне зависимости от того успешно завершилась операция или нет.

Переменной `result` должна быть выделена память и она должна быть инициализирована перед вызовом одной из перечисленных выше функций. Как альтенатива, переменные `result` и `op1` могут быть одной и той же переменной, в таком случае будет выполнена эффективная операция составного присваивания:
```c
zval *a, *b;
MAKE_STD_ZVAL(a);
MAKE_STD_ZVAL(b);

ZVAL_LONG(a, 42);
ZVAL_STRING(b, "3");

/* a += b */
add_function(a, a, b TSRMLS_CC);

php_printf("%Z\n", a); /* 45 */

/* zvals a, b need to be dtored */
```
Некоторые бинарные операторы отсутствуют в списке выше. Например, здесь нет операторов `>` и `>=`. Причина этого в том, что вы можете реализовать их воспользовавшись функциями `using is_smaller_function()` и `is_smaller_or_equal_function()` просто поменяв местами операнды.

Также в списке выше нет функций для реализации операций `&&` и `||`. The reasoning here is that the main feature those operators provide is short-circuiting, which you can’t implement with a simple function. If you take short-circuiting away, both operators are just boolean casts followed by a && or || C-operation.

Кроме бинарных операторов также есть унарные (с одним операндом) функции:
```c
int boolean_not_function(zval *result, zval *op1 TSRMLS_DC); /*  !  */
int bitwise_not_function(zval *result, zval *op1 TSRMLS_DC); /*  ~  */
```
Они работают также как другие функции, но принимают только один операнд. Операции унарного сложения и вычитания отсутствуют, так как они легко могут быть реализованы как `0 + $value` и `0 - $value` соответственно, с помощью функций `add_function()` и `sub_function()`.

Еще 2 функции реализуют операторы `++` и `--`:
```c
int increment_function(zval *op1); /* ++ */
int decrement_function(zval *op1); /* -- */
```
Эти функции не принимают на вход zval `result`, а вместо этого модифицируют переданный операнд. Учтите, что использование этих функций отличается от выполнения `+ 1` или `- 1` с помощью `add_function()`/`sub_function()`. For example incrementing `"a"` will result in `"b"`, but adding `"a" + 1` will result in `1`.

##Сравнения

Все функции сравнения представленные выше выполняют определенные операции, например,
 `is_equal_function()` соответствует `==`, а `is_smaller_function()` выполняет `<`. Альтернативой этим функциям является функция `compare_function()`, которая возвращает более общий результат:
```c
zval *a, *b, *result;
MAKE_STD_ZVAL(a);
MAKE_STD_ZVAL(b);
MAKE_STD_ZVAL(result);

ZVAL_LONG(a, 42);
ZVAL_STRING(b, "24");

compare_function(result, a, b TSRMLS_CC);

if (Z_LVAL_P(result) < 0) {
    php_printf("a is smaller than b\n");
} else if (Z_LVAL_P(result) > 0) {
    php_printf("a is greater than b\n");
} else /*if (Z_LVAL_P(result) == 0)*/ {
    php_printf("a is equal to b\n");
}

/* zvals a, b, result need to be dtored */
```
Функция `compare_function()` установит значение zval-а `result` в одно из значений `-1`, `1` или `0` в случае соотношения между `a` и `b` — “меньше чем”, “больше чем” или “равно” соответственно. Эта функция — часть целого семейства функций сравнения:
```c
int compare_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);

int numeric_compare_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);

int string_compare_function_ex(zval *result, zval *op1, zval *op2, zend_bool case_insensitive TSRMLS_DC);
int string_compare_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);
int string_case_compare_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);

#ifdef HAVE_STRCOLL
int string_locale_compare_function(zval *result, zval *op1, zval *op2 TSRMLS_DC);
#endif
```
Еще раз подчеркнем: все функции принимают на вход 2 операнда, zval `result`, в который будет записан результат и возвращают `SUCCESS`/`FAILURE`.

Функция `compare_function()` выполняет "обычное" в терминах PHP сравнение (то есть ведет себя так же как и операторы `<`, `>` и `==`). Функция `numeric_compare_function()` сравнивает операнды как числа (первым делом приводя их типы к double).

Функций `string_compare_function_ex()` сравнивает операнды как строки и имеет флаг, отвечающий за то, должны ли строки сравниваться без учета регистра букв. Также, вместо того чтобы задавать этот флаг вы можете воспользоваться функциями `string_compare_function()` (регистрозависимое сравнение) или `string_case_compare_function()` (регистронезависимое сравнение). Переменные сравниваются этими функциями как обычные строки без дополнительной магии для чисел представленных как строки.

Функция `string_locale_compare_function()` выполняет сравнение строк в соответствии с текущей локалью, эта функция доступна только если определен макрос `HAVE_STRCOLL`. Поэтому вы должны делать проверку `#ifdef HAVE_STRCOLL` прежде чем использовать эту функцию. Лучше избегать использования этой функции..

##Приведение типов

При реализации своего кода вы часто будете работать с одними и теми же типами zval-ов. Например, если вы пишите код обрабатывающий строки, то вы можете захотеть работать только с строковыми zval-ами и не беспокоиться о всех остальных типах. С другой стороны вы можете захотеть поддерживать систему динамических типов PHP, которая позволяет работать с числами как со строками. Расширениям следует поддерживать систему динамических типов.

Для поддержки системы динамических типов вам следует преобразовывать ваши zval-ы к тем типам, с которыми вы собираетесь работать. Для этой цели PHP предоставляет функции `convert_to_*` для каждого типа (кроме ресурсов, так как не существует тайп кастинга (resource)):
```c
void convert_to_null(zval *op);
void convert_to_boolean(zval *op);
void convert_to_long(zval *op);
void convert_to_double(zval *op);
void convert_to_string(zval *op);
void convert_to_array(zval *op);
void convert_to_object(zval *op);

void convert_to_long_base(zval *op, int base);
void convert_to_cstring(zval *op);
```
Последние две функции предоставляют нестандартные преобразования типов: [`convert_to_long_base()`](https://github.com/php/php-src/blob/44dcdd146d36c91dddc9b89b4679074fdbf2e183/Zend/zend_operators.c#L369) — это то же что и `convert_to_long()`, но эта функция позволяет задать основание для преобразования строки в число (например, 16 для преобразование строки в шестнадцатеричное число). Функция `convert_to_cstring()` ведет себя также как `convert_to_string()`, но использует независимое от локали преобразование переменных типа double в строки, то есть в качестве десятичного разделителя всегда будет использоваться точка, а не запятая (как в немецкой локали).

Функции `convert_to_*` напрямую модифицируют переденный zval:
```c
zval *zv_ptr;
MAKE_STD_ZVAL(zv_ptr);
ZVAL_STRING(zv_ptr, "123 foobar", 1);

convert_to_long(zv_ptr);

php_printf("%ld\n", Z_LVAL_P(zv_ptr));

zval_dtor(&zv_ptr);
```
Если zval используется более чем в одном месте (refcount > 1), то есть шанс что его прямое изменение может привести к некорректному поведению, например, если вы получаете zval по значению и напрямую применяете к нему одну из функций `convert_to_*`, то вы измените не только ссылку на этот zval внутри функции, но и ссылку снаружи.

Чтобы решить эту проблему PHP предоставляет набор дополнительных макросов `convert_to_*_ex macros`:
```c
void convert_to_null_ex(zval **ppzv);
void convert_to_boolean_ex(zval **ppzv);
void convert_to_long_ex(zval **ppzv);
void convert_to_double_ex(zval **ppzv);
void convert_to_string_ex(zval **ppzv);
void convert_to_array_ex(zval **ppzv);
void convert_to_object_ex(zval **ppzv);
```
Эти макросы принимают на вход `zval**` и выполняют `SEPARATE_ZVAL_IF_NOT_REF()` перед преобразованием типов:
```с
#define convert_to_ex_master(ppzv, lower_type, upper_type)  \
    if (Z_TYPE_PP(ppzv)!=IS_##upper_type) {                 \
        SEPARATE_ZVAL_IF_NOT_REF(ppzv);                     \
        convert_to_##lower_type(*ppzv);                     \
    }
```
В остальном эти макросы аналогичны использованию функций `convert_to_*`:
```c
zval **zv_ptr_ptr = /* get function argument */;

convert_to_long_ex(zv_ptr_ptr);

php_printf("%ld\n", Z_LVAL_PP(zv_ptr_ptr));

/* Деструктор не нужен так как аргументы функций уничтожаются автоматически */
```
Но даже этого не всегда достаточно. Представим похожую ситуацию, в которой значение извлекается из массива:
```c
zval *array_zv = /* get array from somewhere */;

/* Fetch array index 42 into zv_dest (how this works is not relevant here) */
zval **zv_dest;
if (zend_hash_index_find(Z_ARRVAL_P(array_zv), 42, (void **) &zv_dest) == FAILURE) {
    /* Error: Index not found */
    return;
}

convert_to_long_ex(zv_dest);

php_printf("%ld\n", Z_LVAL_PP(zv_dest));

/* Деструктор не нужен так как аргументы функций уничтожаются автоматически */
```
Использование `convert_to_long_ex()` в примере выше предотвратит модификацию ссылок на значение снаружи массива, но все равно поменяет значение внутри массива. Иногда это приемлемое поведение, но чаще вы захотите избежать изменения элементовмассива при извлечении значений из него.

В таких случаях нет другого варианта кроме как копировать zval перед преобразованием типа:
```c
zval **zv_dest = /* get array value */;
zval tmp_zv;

ZVAL_COPY_VALUE(&tmp_zv, *zv_dest);
zval_copy_ctor(&tmp_zv);

convert_to_long(&tmp_zv);

php_printf("%ld\n", Z_LVAL(tmp_zv));

zval_dtor(&tmp_zv);
```
Последний вызов `zval_dtor()` в коде выше необязателен, так как мы знаем, что`tmp_zv` будет иметь тип `IS_LONG`, а он не требует обязательного вызова деструктора. Для преобразования других типов, такх как строки или массивы, вызов деструктора необходим.

Если в вашем коде часто встречаются преобразования в тип long или в тип double, то вам имеет смысл создать функции хелперы, которые будут делать преобразование без модификации zval-а. Вот пример такой функции для преобразования в тип long:
```c
long zval_get_long(zval *zv) {
    switch (Z_TYPE_P(zv)) {
        case IS_NULL:
            return 0;
        case IS_BOOL:
        case IS_LONG:
        case IS_RESOURCE:
            return Z_LVAL_P(zv);
        case IS_DOUBLE:
            return zend_dval_to_lval(Z_DVAL_P(zv));
        case IS_STRING:
            return strtol(Z_STRVAL_P(zv), NULL, 10);
        case IS_ARRAY:
            return zend_hash_num_elements(Z_ARRVAL_P(zv)) ? 1 : 0;
        case IS_OBJECT: {
            zval tmp_zv;
            ZVAL_COPY_VALUE(&tmp_zv, zv);
            zval_copy_ctor(&tmp);
            convert_to_long_base(&tmp, 10);
            return Z_LVAL_P(tmp_zv);
        }
    }
}
```
Код выше возвращает результат преобразования не выполняя копирования zval-ов (кроме случая, в котором преобразуется `IS_OBJECT`, здесь копирование неизбежно). С использованием этой функции пример преобразования значения в массиве становится гораздо проще:
```c
zval **zv_dest = /* get array value */;
long lval = zval_get_long(*zv_dest);

php_printf("%ld\n", lval);
```
Стандартная библиотека PHP уже содержит одну функцию такого типа, она называется [`zend_is_true()`](https://github.com/php/php-src/blob/ccf863c8ce7e746948fb060d515960492c41ed27/Zend/zend_execute_API.c#L446). Эта функция является функциональным эквивалентом приобразования к типу bool, в котором значение возвращается напрямую:
```c
zval *zv_ptr;
MAKE_STD_ZVAL(zv_ptr);

ZVAL_STRING(zv, "", 1);
php_printf("%d\n", zend_is_true(zv)); // 0
zval_dtor(zv);

ZVAL_STRING(zv, "foobar", 1);
php_printf("%d\n", zend_is_true(zv)); // 1
zval_ptr_dtor(&zv);
```
Еще одна функция, которая позволяет избежать ненужного копирования при преобразовании типов это [`zend_make_printable_zval()`](https://github.com/php/php-src/blob/149568f4da75a148d6ca71073b353f0d5f8f477a/Zend/zend.c#L226). Эта функция делает такое же преобразование в строку как `convert_to_string()`, но выполняет это используя другой API. Типичное использование:
```c
zval *zv_ptr = /* получаем zval откуда-то еще */;

zval tmp_zval;
int tmp_zval_used;
zend_make_printable_zval(zv_ptr, &tmp_zval, &tmp_zval_used);

if (tmp_zval_used) {
    zv_ptr = &tmp_zval;
}

PHPWRITE(Z_STRVAL_P(zv_ptr), Z_STRLEN_P(zv_ptr));

if (tmp_zval_used) {
    zval_dtor(&tmp_zval);
}
```
Второй аргумент в этой функции — указатель не временный zval, третий — указатель на  integer. Если эта функция использовала внутри себя временный zval, то она установит третий аргумент в единицу, иначе — в ноль.

Основываясь на значении `tmp_zval_used` вы можете решить использовать оригинальный zval или временную копию. Удобно присвоить значение временного zval-а оригинальному при помощи `zv_ptr = &tmp_zval`. Это позволит вам всегда работать с `zv_ptr` вместо того чтобы везде использовать условие для выбора одного из 2 значений.

В конце вам нужно вызвать деструктор для временного zval-а `zval_dtor(&tmp_zval)`, но только если он действительно использовался.

Еще одна функция связанная с преобразованием типов это `is_numeric_string()`. Она проверяет является ли содержимое строки числом и извлекает его значение в член-данных zval-а `lval` или `dval` в зависимости от содержимого строки:
```c
long lval;
double dval;

switch (is_numeric_string(Z_STRVAL_P(zv_ptr), Z_STRLEN_P(zv_ptr), &lval, &dval, 0)) {
    case IS_LONG:
        /* String is an integer those value was put into `lval` */
        break;
    case IS_DOUBLE:
        /* String is a double those value was put into `dval` */
        break;
    default:
        /* String is not numeric */
}
```
Последний аргумент этой функции называется `allow_errors` (разрешить ошибки). Установка его в значение `0` запретит использовать строки вида "123abc", а установка в `1` позволит использовать такие строки (результатом будет 123). Значение `-1` предоставляет среднее решение: строка будет преобразована, но будет выведено предупреждение (notice).

Также полезно знать, что эта функция позволяет работать с шестнадцатеричными числами в формате `0xabc`. Это её отличие от `convert_to_long()` и `convert_to_double()`, которые преобразуют `"0xabc"` в ноль.

Функция `is_numeric_string()` особенно удобна в случае когда вы можете работать одновременно и с типом integer, и с типом double, но вы не хотите нести потери точности связанные с использованием double в обоих случаях. Еще одна вспомогательная функция [`convert_scalar_to_number()`](https://github.com/php/php-src/blob/dc8a53d4baeb2a4c97b367b78b3d47b53fe6ecbb/Zend/zend_operators.c#L184), которая принимает zval и конвертирует значения не-массивы в `long` или `double` (используя `is_numeric_string()` для строк). Это значит, что сконвертированный zval будет иметь тип `IS_LONG`, `IS_DOUBLE` или `IS_ARRAY`. Использовать эту функцию можно также как `convert_to_*()`:
```c
zval *zv_ptr;
MAKE_STD_ZVAL(zv_ptr);
ZVAL_STRING(zv_ptr, "3.141", 1);

convert_scalar_to_number(zv_ptr);
switch (Z_TYPE_P(zv_ptr)) {
    case IS_LONG:
        php_printf("Long: %ld\n", Z_LVAL_P(zv_ptr));
        break;
    case IS_DOUBLE:
        php_printf("Double: %G\n", Z_DVAL_P(zv_ptr));
        break;
    case IS_ARRAY:
        /* Likely throw an error here */
        break;
}

zval_ptr_dtor(&zv_ptr);

/* Double: 3.141 */
```
Также существует вариант этой функции `convert_scalar_to_number_ex()`, который принимает `zval**` и разделяет его перед преобразованием.
