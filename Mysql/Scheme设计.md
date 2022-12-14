# 数据类型的选择

1. 更小的通常更好
2. 简单就好
3. 尽量避免 Null

`InnoDB` 使用单独的位存储 Null 值，所以对于稀疏数据（多数为 Null，少数非 Null）有很好的空间效率。

MySQL 很多数据类型只是别名，可以用 `SHOW CREATE TABLE` 查看对应的基本类型。

## 整数

整数类型： `TINYINT` 、 `SMALLINT` 、 `MEDIUMINT` 、 `INT` 、 `BIGINT`；分别使用 8、16、24、32、64 位存储空间。存储的范围从 -2(N-1) 到 2(N-1)-1。

整数类型有可选的 `UNSIGNED`，表示不允许负值。

有符号和无符号类型使用相同的存储空间（无符号数不能表示负数，但能表示更多的正整数），并具有相同的性能，因此可以根据实际情况选择合适的类型。

MySQL 可以为整数类型指定宽度，例如 `INT(11)`，这实际没有意义：它不会限制值的合法范围。对于存储和计算来说， `INT(1)` 和 `INT(20)` 是相同的。

## 实数

`DECIMAL` 类型用于存储精确的小数。CPU 不支持对 `DECIMAL` 的直接计算。

CPU 直接支持原生浮点计算，所以浮点运算明显更快。

MySQL 5.0 和更高版本中的 `DECIMAL` 类型最多可以包含 65 个数字。

定义数据类型为`DECIMAL`的列的语法：

```mysql
column_name DECIMAL(P,D);
```

- `P`是表示有效数字数的精度。 `P`范围为`1〜65`。
- `D`是表示小数点后的位数。 `D`的范围是`0`~`30`。MySQL要求`D <= P`。
- `(P,D)`需要存储 m+2 个字节。

```mysql
amount DECIMAL(6,2);
```

在此示例中，`amount`列最多可以存储`6`位数字，小数位数为`2`位; 因此，`amount`列的范围是从`-9999.99`到`9999.99`。

浮点类型在存储同样范围的值时，通常比 `DECIMAL` 使用更少的空间。 `FLOAT` 使用 4 个字节存储； `DOUBLE` 占用 8 个字节。

MySQL 使用 `DOUBLE` 作为内部浮点计算的类型。

因为需要额外的空间和计算开销，所以应该尽量只在对小数进行精确计算时才使用 `DECIMAL` 。

在数据量比较大的时候，可以考虑使用 `BIGINT` 代替 `DECIMAL` ，将需要存储的货币单位根据小数的位数乘以相应的倍数即可。

## 字符串类型

### VARCHAR

VARCHAR 用于存储可变长度字符串，比定长类型更节省空间。

`VARCHAR` 需要使用 1 或 2个额外字节记录字符串的长度：如果列的最大长度小于或者等于255字节，则只使用1个字节表示，否则使用 2 个字节。

`VARCHAR` 节省了存储空间，所以对性能也有帮助。但是，行变长时，如果页内没有更多的空间可以存储，`MyISAM` 会将行拆成不同的片段存储，`InnoDB` 则需要分裂页来使行可以放进页内。

### CHAR

根据定义分配足够的空间。当存储 `CHAR` 值时，MySQL 会删除所有的末尾空格。`CHAR` 值会根据需要采用空格进行填充以方便比较。

- `CHAR` 适合存储很短的字符串，或者所有值都接近同一个长度，比如密码的 MD5 值。
- 对于经常变更的数据， `CHAR` 也比 `VARCHAR` 更好，定长不容易产生碎片。
- 非常短的列， `CHAR` 比 `VARCHAR` 在存储空间上更有效率。

### BLOB 和 TEXT

`BLOB` 和 `TEXT` 都是为存储很大的数据而设计的字符串数据类型，分别采用二进制和字符串方式存储。

字符串类型： `TINYTEXT`、 `SMALLTEXT`、 `TEXT`、 `MEDIUMTEXT`、 `LONGTEXT`
二进制类型： `TINYBLOB`、 `SMALLBLOB`、 `BLOB`、 `MEDIUMBLOB`、 `LONGBLOB`

`BLOB` 是 `SMALLBLOB` 的同义词； `TEXT` 是 `SMALLTEXT` 的同义词。

MySQL 把每个 `BLOB` 和 `TEXT` 值当做一个独立的对象处理。InnoDB 会使用专门的“外部”存储区域来进行存储，此时每个值在行内需要 1 ~ 4 个字节存储一个指针，然后在外部存储区域存储实际的值。

`BLOB` 和 `TEXT` 家族之间仅有的不同是 `BLOB` 类型存储的是二进制，没有排序规则或字符集，而 `TEXT` 类型有字符集和排序规则。

`BLOB` 和 `TEXT` 只对每个列的最前 `max_sort_length` 字节而不是整个字符串做排序。

### ENUM

枚举列可以把一些不重复的字符串存储成一个预定义的集合。

MySQL 在存储枚举时非常紧凑，会根据列表值的数量压缩到一个或者两个字节中。MySQL 在内部会将每个值在列表中的位置保存为整数，并且在表的`.frm`文件中保存 “数字-字符串” 映射关系的 “查找表”。

## 日期与时间类型

MySQL 能存储的最小时间粒度为秒。但，也可以使用微秒级的粒度进行临时运算。

- `DATETIME`

  保存大范围的值，从 1001 年到 9999 年，精度为秒。把日期和时间封装到格式为 YYYYMMDDHHMMSS 的整数中，与时区无关。使用 8 个字节的存储空间。

- `TIMESTAMP`

  保存从 1970 年 1 月 1 日午夜以来的秒数，和 UNIX 时间戳相同。`TIMESTAMP` 只使用 4 个字节的存储空间，范围是从 1970 年到 2038 年。

默认情况下，如果插入时没有指定第一个 `TIMESTAMP` 列的值，MySQL 则设置这个列的值为当前时间。

`TIMESTAMP` 列默认为 `NOT NULL`。

通常应该尽量使用 `TIMESTAMP` ，因为它比 `DATETIME` 空间效率更高。

可以使用 `BIGINT` 类型存储微秒级别的时间戳，或者使用 `DOUBLE` 存储秒之后的小数部分。

更有可能使用标识列与其他值进行比较，或者通过标识列寻找其他列。

## 主键类型选择

选择标识列的类型时，不仅仅需要**考虑存储类型**，还需要**考虑 MySQL 对这种类型怎么执行计算和比较**。

一旦选定一种类型，要确保在所有关联表中都使用同样的类型。类型之间需要精确匹配，包括像 `UNSIGNED` 这样的属性。混用不同数据类型可能导致性能问题，在比较操作时隐式类型转换也可能导致很难发现的错误。

- 整数类型

  整数通常是标识列最好的选择，因为它们很快并且可以使用 `AUTO_INCREMENT`。

- `ENUM` 和 `SET` 类型

  通常是一个糟糕的选择。 `ENUM` 和 `SET` 列适合存储固定信息。

- 字符串类型

  如果可能，应该避免使用字符串作为标识列，因为它们很消耗空间，并且通常比数字类型慢。`MyISAM` 默认对字符串使用压缩索引，这会导致查询慢很多。使用完全“随机”的字符串也需要多加注意，例如 MD5()、SHA1()、 UUID()产生的字符串。这些新值会任意分布在很大的空间内，这会导致 `INSERT` 以及一些 `SELECT` 语句变得很慢：插入值会随机地写到索引的不同位置，所以使得 `INSERT` 语句更慢。这会导致页分裂、磁盘随机访问，以及对于聚簇存储引擎产生聚簇索引碎片。`SELECT` 语句会变得更慢，因为逻辑上相邻的行会分布在磁盘和内存的不同地方。随机值导致缓存对所有类型的查询语句效果都很差，因为会使得缓存赖以工作的局部访问性原理失效。如果真个数据集都一样的“热”，那么缓存任何一部分特别数据到内存都没有好处；如果工作集比内存大，缓存将会有很多刷新和不命中。

如果存储 UUID 值，则应该移除 “-” 符号；更好的做法是，使用 `UNHEX()` 函数转换 UUID 值为 16 字节的数字，并且存储在一个 `BINARY(16)` 列中。检索时可以通过 `HEX()`函数来格式化为十六进制格式。

# 范式与反范式

第一范式：符合1NF的关系中的每个属性都不可再分。1NF是所有关系型数据库的最基本要求。

**范式化通常带来的好处：**

- 范式化的更新操作通常比反范式化要快。
- 当数据较好地范式化时，就只有很少或者没有重复数据，所以只需要修改更少的数据。
- 范式化的表通常更小，可以更好地存放在内存里，所以执行操作会更快。
- 很少有多余的数据意味着检索列表数据时，更少需要 `DISTINCT` 或者 `GROUP BY` 语句。

范式化设计的 Schema 的缺点是通常需要关联。

**反范式的优缺点**

- 反范式化的 Schema 因为所有数据都在一张表中，可以很好地避免关联。
- 单独的表也能使用更有效的索引策略。

