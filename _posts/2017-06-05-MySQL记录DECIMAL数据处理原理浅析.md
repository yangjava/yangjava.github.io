---
layout: post
categories: [MySQL]
description: none
keywords: MySQL
---
# MySQL记录DECIMAL 数据处理原理浅析

文章开始前先复习一下官方文档关于 DECIMAL 类型的一些介绍：
```
The declaration syntax for a DECIMAL column is DECIMAL(M,D). The ranges of values for the arguments are as follows:

M is the maximum number of digits (the precision). It has a range of 1 to 65.

D is the number of digits to the right of the decimal point (the scale). It has a range of 0 to 30 and must be no larger than M.

If D is omitted, the default is 0. If M is omitted, the default is 10.

The maximum value of 65 for M means that calculations on DECIMAL values are accurate up to 65 digits. This limit of 65 digits of precision also applies to exact-value numeric literals, so the maximum range of such literals differs from before. (There is also a limit on how long the text of DECIMAL literals can be; see Section 12.25.3, “Expression Handling”.)
```
以上材料提到的最大精度和小数位是本文分析关注的重点：

最大精度是 65 位
小数位最多 30 位
接下来将先分析 MySQL 服务输入处理 DECIMAL 类型的常数。

现在，先抛出几个问题：

MySQL 中当使用 SELECT 查询常数时，例如：SELECT 123456789.123; 是如何处理的？
MySQL 中查询一下两条语句分别返回结果是多少？为什么？
```
SELECT 111111111111111111111111111111111111111111111111111111111111111111111111111111111;
SELECT 1111111111111111111111111111111111111111111111111111111111111111111111111111111111;
```

## MySQL 如何解析常数
来看第1个问题，MySQL 的词法分析在处理 SELECT 查询常数的语句时，会根据数字串的长度选择合适的类型来存储数值，决策逻辑代码位于 int_token(const char *str, uint length)@sql_lex.cc，具体的代码片段如下：
```
static inline uint int_token(const char *str, uint length) {
  ...
  if (neg) {
      cmp = signed_long_str + 1;
      smaller = NUM;      // If <= signed_long_str
      bigger = LONG_NUM;  // If >= signed_long_str
    } else if (length < signed_longlong_len)
      return LONG_NUM;
    else if (length > signed_longlong_len)
      return DECIMAL_NUM;
    else {
      cmp = signed_longlong_str + 1;
      smaller = LONG_NUM;  // If <= signed_longlong_str
      bigger = DECIMAL_NUM;
    }
  } else {
    if (length == long_len) {
      cmp = long_str;
      smaller = NUM;
      bigger = LONG_NUM;
    } else if (length < longlong_len)
      return LONG_NUM;
    else if (length > longlong_len) {
      if (length > unsigned_longlong_len) return DECIMAL_NUM;
      cmp = unsigned_longlong_str;
      smaller = ULONGLONG_NUM;
      bigger = DECIMAL_NUM;
    } else {
      cmp = longlong_str;
      smaller = LONG_NUM;
      bigger = ULONGLONG_NUM;
    }
  }
  while (*cmp && *cmp++ == *str++)
    ;
  return ((uchar)str[-1] <= (uchar)cmp[-1]) ? smaller : bigger;
}
```
上面代码中，long_len 值为 10，longlong_len 值为 19，unsigned_longlong_len值为20。

neg表示是否是负数，直接看正数的处理分支，负数同理：

当输入的数值串长度等于 10 时 MySQL 可能使用 LONG_NUM 或 LONG_NUM 表示
当输入的数值串长度小于 19 时 MySQL 使用 LONG_NUM 表示
当输入的数值串长度等于 20 时 MySQL 可能使用 LONG_NUM 或 DECIMAL_NUM 表示
当输入的数值串长度大于 20 时 MySQL 使用 DECIMAL_NUM 表示
其他长度时，MySQL 可能使用 LONG_NUM或ULONGLONG_NUM 表示
对于可能有两种表示方式的数据，MySQL 是通过将数字串与 cmp 指向的数值字符串进行比较，如果小于等于 cmp 表示的数值则使用 smaller 表示，否则使用 bigger 表示。cmp 指向的数值字符串定义在 sql_lex.cc 文件中，具体如下：
```
static const char *long_str = "2147483647";
static const uint long_len = 10;
static const char *signed_long_str = "-2147483648";
static const char *longlong_str = "9223372036854775807";
static const uint longlong_len = 19;
static const char *signed_longlong_str = "-9223372036854775808";
static const uint signed_longlong_len = 19;
static const char *unsigned_longlong_str = "18446744073709551615";
static const uint unsigned_longlong_len = 20;
```
因此，这里我们可以得出结论：MySQL 中当使用 SELECT 查询常数时，根据数值串的长度和数值大小来决定使用什么类型来接收常数。当数值串长度大于 20，或数值串长度等于 20 且数值小于-9223372036854775808或大于18446744073709551615时，MySQL 服务选择使用 DECIMAL 类型来接收处理常数。

这里，再抛出一个问题：

上面分析提到的 DECIMAL 是否与官方文档中提到的 DECIMAL 类型或者换一种方式说：是否与建表语句 CREATE TABLE t(d DECIMAL(65, 30)); 中字段 d 的 DECIMAL(65, 30)类型(可以不考虑精度和小数位)相同？

## MySQL 解析 DECIMAL 常数时怎么处理溢出
分析第2个问题，先看一下语句的执行结果：
```
root@mysqldb 14:09:  [(none)]> SELECT 111111111111111111111111111111111111111111111111111111111111111111111111111111111;
+-----------------------------------------------------------------------------------+
| 111111111111111111111111111111111111111111111111111111111111111111111111111111111 |
+-----------------------------------------------------------------------------------+
| 111111111111111111111111111111111111111111111111111111111111111111111111111111111 |
+-----------------------------------------------------------------------------------+
1 row in set (2.28 sec)

root@mysqldb 14:09:  [(none)]> SELECT 1111111111111111111111111111111111111111111111111111111111111111111111111111111111;
+------------------------------------------------------------------------------------+
| 1111111111111111111111111111111111111111111111111111111111111111111111111111111111 |
+------------------------------------------------------------------------------------+
|                  99999999999999999999999999999999999999999999999999999999999999999 |
+------------------------------------------------------------------------------------+
1 row in set, 1 warning (2.01 sec)
```
接着上面的思路往下看常数的语法解析：
```
NUM_literal:
          int64_literal
        | DECIMAL_NUM
          {
            $$= NEW_PTN Item_decimal(@$, $1.str, $1.length, YYCSCL);
          }
        | FLOAT_NUM
          {
            $$= NEW_PTN Item_float(@$, $1.str, $1.length);
          }
        ;
```
语法解析器在获取到 toekn = DECIMAL_NUM 后，会创建一个 Item_decimal 对象来存储输入的数值。

在分析代码之前先来看几个常数定义：
```
/** maximum length of buffer in our big digits (uint32). */
static constexpr int DECIMAL_BUFF_LENGTH{9};

/** the number of digits that my_decimal can possibly contain */
static constexpr int DECIMAL_MAX_POSSIBLE_PRECISION{DECIMAL_BUFF_LENGTH * 9};

/**
  maximum guaranteed precision of number in decimal digits (number of our
  digits * number of decimal digits in one our big digit - number of decimal
  digits in one our big digit decreased by 1 (because we always put decimal
  point on the border of our big digits))
*/
static constexpr int DECIMAL_MAX_PRECISION{DECIMAL_MAX_POSSIBLE_PRECISION -
                                           8 * 2};

static constexpr int DECIMAL_MAX_SCALE{30};
```
DECIMAL_BUFF_LENGTH：表示整个 DECIMAL 类型数据的缓冲区大小
DECIMAL_MAX_POSSIBLE_PRECISION：每个缓冲区单元可以存储 9 位数字，所以最大可以处理的精度这里为 81
DECIMAL_MAX_PRECISION：用来限制官方文档介绍中 decimal(M,D) 中的 M 的最大值，亦或是当超大常数溢出后返回的整数部分最大长度
DECIMAL_MAX_SCALE：用来限制官方文档介绍中 decimal(M,D) 中的 D 的最大值
```
Item_decimal::Item_decimal(const POS &pos, const char *str_arg, uint length,
                           const CHARSET_INFO *charset)
    : super(pos) {
  str2my_decimal(E_DEC_FATAL_ERROR, str_arg, length, charset, &decimal_value);
  item_name.set(str_arg);
  set_data_type(MYSQL_TYPE_NEWDECIMAL);
  decimals = (uint8)decimal_value.frac;
  fixed = true;
  max_length = my_decimal_precision_to_length_no_truncation(
      decimal_value.intg + decimals, decimals, unsigned_flag);
}
```
在Item_decimal构造函数中调用str2my_decimal函数对输入数值进行处理，将其转换为my_decimal类型的数据。
```
int str2my_decimal(uint mask, const char *from, size_t length,
                   const CHARSET_INFO *charset, my_decimal *decimal_value) {
  const char *end, *from_end;
  int err;
  char buff[STRING_BUFFER_USUAL_SIZE];
  String tmp(buff, sizeof(buff), &my_charset_bin);
  if (charset->mbminlen > 1) {
    uint dummy_errors;
    tmp.copy(from, length, charset, &my_charset_latin1, &dummy_errors);
    from = tmp.ptr();
    length = tmp.length();
    charset = &my_charset_bin;
  }
  from_end = end = from + length;
  err = string2decimal(from, (decimal_t *)decimal_value, &end);
  if (end != from_end && !err) {
    /* Give warning if there is something other than end space */
    for (; end < from_end; end++) {
      if (!my_isspace(&my_charset_latin1, *end)) {
        err = E_DEC_TRUNCATED;
        break;
      }
  }
  check_result_and_overflow(mask, err, decimal_value);
  return err;
}
```
str2my_decimal 函数先将数值字符串转为合适的字符集后，调用 string2decimal 函数将数值字符串转为 decimal_t 类型的数据。my_decimal 类型和 decimal_t 类型的关系如下：
```
@startuml

class decimal_t 
{
  + int intg, frac, len;
  + bool sign;
  + decimal_digit_t *buf;
}

class my_decimal
{
  - decimal_digit_t buffer[DECIMAL_BUFF_LENGTH];
}

decimal_t <|-- my_decimal
@enduml
```
decimal_digit_t 是 int32_t 的别名
intg 表示整数部分的字符个数
frac 表示小数部分的字符个数
sign 表示是否负数
buf 指向 buffer
buffer 是数据存放数组，数组长度为9，也就意味着一个 decimal 最多可以存放 9 个 int32_t 大小的数据，但由于设计限制每个数组元素限制存储 9 个字符，因此 buffer 最多可以存储81个字符

由于 buffer 长度的限制，在 string2decimal 函数解析时会有溢出的可能，因此，解析后还需要调用check_result_and_overflow函数处理溢出的情况。

string2decimal 的代码实现：
```
int string2decimal(const char *from, decimal_t *to, const char **end) {
  const char *s = from, *s1, *endp, *end_of_string = *end;
  int i, intg, frac, error, intg1, frac1;
  dec1 x, *buf;
  sanity(to);

  error = E_DEC_BAD_NUM; /* In case of bad number */
  while (s < end_of_string && my_isspace(&my_charset_latin1, *s)) s++;
  if (s == end_of_string) goto fatal_error;

  if ((to->sign = (*s == '-')))
    s++;
  else if (*s == '+')
    s++;

  s1 = s;
  while (s < end_of_string && my_isdigit(&my_charset_latin1, *s)) s++;
  intg = (int)(s - s1);
  if (s < end_of_string && *s == '.') {
    endp = s + 1;
    while (endp < end_of_string && my_isdigit(&my_charset_latin1, *endp))
      endp++;
    frac = (int)(endp - s - 1);
  } else {
    frac = 0;
    endp = s;
  }

  *end = endp;

  if (frac + intg == 0) goto fatal_error;

  error = 0;

  intg1 = ROUND_UP(intg);
  frac1 = ROUND_UP(frac);
  FIX_INTG_FRAC_ERROR(to->len, intg1, frac1, error);
  if (unlikely(error)) {
    frac = frac1 * DIG_PER_DEC1;
    if (error == E_DEC_OVERFLOW) intg = intg1 * DIG_PER_DEC1;
  }

  /* Error is guranteed to be set here */
  to->intg = intg;
  to->frac = frac;

  buf = to->buf + intg1;
  s1 = s;

  for (x = 0, i = 0; intg; intg--) {
    x += (*--s - '0') * powers10[i];

    if (unlikely(++i == DIG_PER_DEC1)) {
      *--buf = x;
      x = 0;
      i = 0;
    }
  }
  if (i) *--buf = x;

  buf = to->buf + intg1;
  for (x = 0, i = 0; frac; frac--) {
    x = (*++s1 - '0') + x * 10;

    if (unlikely(++i == DIG_PER_DEC1)) {
      *buf++ = x;
      x = 0;
      i = 0;
    }
  }
  if (i) *buf = x * powers10[DIG_PER_DEC1 - i];

  /* Handle exponent */
  if (endp + 1 < end_of_string && (*endp == 'e' || *endp == 'E')) {
    int str_error;
    longlong exponent = my_strtoll10(endp + 1, &end_of_string, &str_error);

    if (end_of_string != endp + 1) /* If at least one digit */
    {
      *end = end_of_string;
      if (str_error > 0) {
        error = E_DEC_BAD_NUM;
        goto fatal_error;
      }
      if (exponent > INT_MAX / 2 || (str_error == 0 && exponent < 0)) {
        error = E_DEC_OVERFLOW;
        goto fatal_error;
      }
      if (exponent < INT_MIN / 2 && error != E_DEC_OVERFLOW) {
        error = E_DEC_TRUNCATED;
        goto fatal_error;
      }
      if (error != E_DEC_OVERFLOW) error = decimal_shift(to, (int)exponent);
    }
  }
  /* Avoid returning negative zero, cfr. decimal_cmp() */
  if (to->sign && decimal_is_zero(to)) to->sign = false;
  return error;

fatal_error:
  decimal_make_zero(to);
  return error;
}
```
解析过程大致如下：

分别计算整数部分和小数部分各有多少个字符
分别计算整数部分和小数部分各需要多少个 buffer 元素来存储

如果整数部分需要的 buffer 元素个数超过 9，则表示溢出
如果整数部分和小数部分需要的 buffer 元素个数超过 9，则表示需要将小数部分进行截断
由于先解析整数部分，再解析小数部分，因此，如果整数部分如果完全占用所有 buffer 元素，此时，小数部分会被截断。
将整数部分和小数部分按每 9 个字符转为一个整数记录到 buffer 的元素中（buffer中的模型示例如下）
```
例如常数：111111111222222222333333333.444444444

intg = 27, frac = 9, len = 9, sign = false
byte         0            1           2           3          4         5         6         6         7         8
buffer: | 111111111 | 222222222 | 333333333 | 444444444 | UNKNOWN | UNKNOWN | UNKNOWN | UNKNOWN | UNKNOWN | UNKNOWN |
     低地址  -----------------------------------------------------------------------------------------------> 高地址
```
check_result_and_overflow 代码实现:
```
void max_decimal(int precision, int frac, decimal_t *to) {
  int intpart;
  dec1 *buf = to->buf;
  assert(precision && precision >= frac);

  to->sign = false;
  // 发生溢出时将 buffer 中的数据更新为 9 99 999 ...
  if ((intpart = to->intg = (precision - frac))) {
    int firstdigits = intpart % DIG_PER_DEC1;
    if (firstdigits) *buf++ = powers10[firstdigits] - 1; /* get 9 99 999 ... */
    for (intpart /= DIG_PER_DEC1; intpart; intpart--) *buf++ = DIG_MAX;
  }

  if ((to->frac = frac)) {
    int lastdigits = frac % DIG_PER_DEC1;
    for (frac /= DIG_PER_DEC1; frac; frac--) *buf++ = DIG_MAX;
    if (lastdigits) *buf = frac_max[lastdigits - 1];
  }
}

inline void max_my_decimal(my_decimal *to, int precision, int frac) {
  assert((precision <= DECIMAL_MAX_PRECISION) && (frac <= DECIMAL_MAX_SCALE));
  max_decimal(precision, frac, to);
}

inline void max_internal_decimal(my_decimal *to) {
  max_my_decimal(to, DECIMAL_MAX_PRECISION, 0);
}

inline int check_result_and_overflow(uint mask, int result, my_decimal *val) {
  // 检查前面的处理是否发生溢出
  if (val->check_result(mask, result) & E_DEC_OVERFLOW) {
    bool sign = val->sign();
    val->sanity_check();
    max_internal_decimal(val);
    val->sign(sign);
  }
  /*
    Avoid returning negative zero, cfr. decimal_cmp()
    For result == E_DEC_DIV_ZERO *val has not been assigned.
  */
  if (result != E_DEC_DIV_ZERO && val->sign() && decimal_is_zero(val))
    val->sign(false);
  return result;
}
```
如果 check_result_and_overflow 调用之前的处理发生了溢出行为，则意味着 decimal 不能存储完整的数据，MySQL 决定这种情况下仅返回decimal 默认的最大精度数值，由上面的代码片段可以看出最大精度数值是 65 个 9。

超大常量数据生成的 DECIMAL 数据与 DECIMAL 字段类型的区别

通过上面对超大常量数据生成的 DECIMAL 数据处理的分析，可以得出问题3的答案：两者不同，区别如下：

DECIMAL 字段类型有显式的精度和小数位的限制，也就是 DECIMAL 字段插入数据时能插入的正数部分的长度为 M-D，而超大常量数据生成的 DECIMAL 数据则会隐含的优先处理考虑整数部分，整数部分处理完才继续处理小数部分，如果缓冲区不够则将小数位截断，如果缓冲区不够整数部分存放则转为 65 个 9。
在 MySQL 的服务源码中 DECIMAL 字段类型使用 Field_new_decimal 类型接收处理，而超大常量数据生成的 DECIMAL 数据由 Item_decimal 类型接收处理。