## SELECT * 的查询效率问题

![](https://files.mdnice.com/user/19026/15064a26-63e6-45e4-9d3b-5c4b162bed37.png)

《阿里巴巴 Java 开发手册》中，强调：
> 在表查询中，一律不要使用 * 作为查询的字段列表，需要哪些字段必须明确写明

对于这一点，其给出的原因为：
1. 增加查询分析器解析成本
2. 增减字段容易与 resultMap 配置不一致
3. 无用字段增加网络消耗，尤其是 text 类型的字段

### 1. 增加查询分析器解析成本

借这篇文章的机会，回顾一下 MySQL 的客户端架构：

![](https://files.mdnice.com/user/19026/aeceb01e-4373-4523-ab07-1b73e14499f9.png)

接下来，我们着重看一下分析器与优化器这两大模块。

#### 分析器

分析器主要的功能有两个：

- 词法分析
- 语法分析

词法分析就是对 SQL 这一长串字符串进行识别，譬如对 `select ID from T` 来说，分析器会将 `select` 字段识别出来，进而识别该 SQL 是一条查询语句；将 `ID` 识别为列名；将 `T` 识别为表名等等。

而语法分析就是该 SQL 是否满足 SQL 语法规定，如果书写的 SQL 有问题，其便会返回 `You have an error in your SQL syntax` 的提示。

简单来说，分析器的功能就是判断你写的 SQL 是否满足 MySQL 的语法规则，并且弄清楚你这条 SQL 语句到底在做什么。

《阿里巴巴 Java 开发手册》中提到了，`select * from T` 会增加查询分析器解析成本。当执行 `select *` 时，分析器会额外地通过一个查询来获取表的所有列，而当使用 `select` 加所有列名则不会。在高事务的环境下，这种差异可能成为一个明显的开销，但如果仅针对普通的查询来讲，这种开销几乎可以忽略不计，并且有人也做过相关测试，`select *` 和 `select` 所有字段，二者速度上几乎无任何差异；所以，`select *` 虽然会增加查询分析器解析成本，但是对查询性能的影响非常小。

#### 优化器

顾名思义，优化器的功能就是对我们的 SQL 语句进行优化，譬如，我们的表中有多个索引，优化器会决定优先使用哪一个索引，又或者一个查询语句涉及到多表关联时，优化器会决定各个表的连接顺序，找出最佳的执行方案。

这里面涉及到一个很重要的问题是：如果我们使用 `select *` 来做语句查询，那么就失去了优化器让 MySQL **只通过索引来访问数据**的能力。

MySQL InnoDB 存储引擎中，索引使用的数据结构为 B+ 树：

![](https://files.mdnice.com/user/19026/e0246aa0-ca01-419a-a7d4-47c7f4a60e60.png)

B+ 树只有叶子节点（data 域）会存储数据，其他的非叶子节点存储的是索引以及指针；而访问 data 域则意味着增加磁盘 IO 开销。有一种优化 SQL 查询的方法便是只通过索引返回数据，而不需要访问表，即访问 data 域。

举个例子：

- 假设，我们有一个包含 10 列字段的表 `T`
- 在表 `T` 上有一个复合索引（`col1`,`col2`）

如果我们需要查询 `col1` 为 "123"，并获取 `col1`,`col2` 两个字段的信息，便可以写出一条查询 SQL：
```sql
select col1,col2 from T where col1 = '123'
```
因为优化器拥有索引的全部信息，那么就不需要再去访问表，即访问 data 域来查询，我们只通过查询索引就可以返回结果，这便减少了磁盘 IO 开销，大大优化了查询性能。

反之，如果我们写出了：
```sql
select * from T where col1 = '123'
```
这样的查询。那么便永远地失去了让优化器只通过查询索引来返回结果的可能。

### 2. 增减字段容易与 resultMap 配置不一致

如果我们使用 `select col1,col2...` 这种方式，在阅读程序时，我们可以清晰地看到获取的列名，而如果我们使用 `select *`，当开发人员阅读代码时，并不知道这条 SQL 究竟取到了哪些具体的字段，代码的可读性就会变差。

当表结构发生变动，譬如新增某些字段或删除某些字段时，使用 `select col1,col2...` 这种方式，我们也可以很快地修改 mapper 中的查询 SQL，以及 resultMap 对应的字段。

反之，如果我们使用 `select *`，并且 resultMap 中的字段并没有与表的所有列对应时，修改便很容易出错。

### 3. 无用字段增加网络消耗，尤其是 text 类型的字段

在我们并不需要某些字段，并使用了 `select *` 做查询，尤其在返回的无用字段是 BLOB,TEXT 类型字段时，无疑对性能是一种严重的消耗。

BLOB 是一个二进制字符串，主要用于存储二进制大对象，例如图片和音视频；TEXT 则是为了存储大量字符串，譬如文章而设计的字符串数据类型。

当我们使用 `select *` 查询，查询结果中一旦包含了 BLOB 或 TEXT 列时，就会导致服务器读取磁盘上的表做查询，而不是内存中的临时表。其原因自然是可想而知的，MySQL 不可能将 BLOB 和 TEXT 这样的大对象存放于内存中，而是在行内存储一个指针，然后指针指向磁盘中的存储区域。而一旦访问磁盘获取数据就意味着消耗磁盘 IO，意味着性能的下降。

MySQL 官方文档也强调了，仅在确实需要查询结果中包含 BLOG 和 TEXT 列，否则请避免使用 `select *`。

> Instances of BLOB or TEXT columns in the result of a query that is processed using a temporary table causes the server to use a table on disk rather than in memory because the MEMORY storage engine does not support those data types (see Section 8.4.4, “Internal Temporary Table Use in MySQL”). Use of disk incurs a performance penalty, so include BLOB or TEXT columns in the query result only if they are really needed. For example, avoid using SELECT *, which selects all columns.


传输这些大文本字段更是耗费时间的，尤其是当数据库与应用程序不在同一台服务器时，这种网络开销将会非常的大。大量的数据意味着使用更多的网络带宽，网络带宽的增加还意味着数据将需要更长的时间才能到达客户端应用程序。

无用的大数据传输导致造成额外的网络开销，这也是禁用 `select *` 的最重要一点原因。

### 4. 总结

综上所述，禁用 `select *` 的原因可以分为两方面：

1. 性能
2. 代码可维护性

- 导致性能方面的主要原因有：

  - 增加查询分析器解析成本（性能影响很小）
  - `select *` 导致 MySQL 失去了只通过索引来访问数据的能力
  - 返回结果集中的无用大对象 譬如 BLOG，TEXT 类型字段会增加磁盘 IO，网络 IO

- 导致代码可维护性方面的主要原因有：

  - 阅读性差
  - 增减字段容易与 resultMap 配置不一致
  - 等等...
  
 
至此为止，这篇文章就到这里了。欢迎大家关注我的公众号，在这里希望你可以收获更多的知识，我们下一期再见！
 