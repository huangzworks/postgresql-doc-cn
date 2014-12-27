JSON 类型
============

PostgreSQL 的 JSON 数据类型用于储存 `RFC 7159 <http://rfc7159.net/rfc7159>`_ 里面描述的 JSON （JavaScript Object Notation）数据。
虽然 ``text`` 数据类型也可以用来储存 JSON 数据，
但 JSON 数据类型的优势在于它会根据 JSON 规则来强制要求每个被储存的值都是合法的 JSON 数据。
本文档的 `9.15 节 <http://www.postgresql.org/docs/9.4/interactive/functions-json.html>`_\ 展示了各式各样专门用来处理 JSON 类型数据的函数和操作符。

JSON 数据类型分为 ``json`` 数据类型和 ``jsonb`` 数据类型，
这两种数据类型能够接受的输入\ **几乎完全相同**\ ，
它们的主要区别在于性能不同：

- ``json`` 数据类型储存的是输入文本的精确拷贝，
  处理函数在每次执行的时候，
  都必须对这些文本重新进行分析（reparse）。

- 而 ``jsonb`` 数据类型则以无压缩（decomposed）二进制格式来储存数据，
  因为格式转换带来的花销，
  这种类型在处理输入的时候速度会稍微慢一些，
  但是因为这种类型的数据并不需要重新进行分析，
  所以这种数据的处理速度会明显地快很多。
  另外 ``jsonb`` 还支持索引特性，
  这也是一个明显的优点。

因为 ``json`` 类型储存的是输入文本的精确拷贝，
所以它会保留介于各个 token 之间与语义无关的空白，
以及各个键在 JSON 对象之内的排列顺序。
另外，
即使一个 JSON 对象包含多个具有相同键名的值，
它的所有键值对也会被保留，
但是处理函数只会把同名键中的最后一个值作为操作的对象。

与此相反，
``jsonb`` 不会保留任何无关的空白，
不会保留对象键的排列顺序，
也不会保留任何重复的对象键。
如果输入里面指定了重复的键，
那么只有最后一个值会被保留。

一般情况下，
除非有特别的要求（比如针对对象键排列顺序的遗留假设，legacy assumption），
否则的话，
大多数应用程序都应该优先使用 ``jsonb`` 类型来储存 JSON 数据，

..
    PostgreSQL allows only one character set encoding per database. 
    It is therefore not possible for the JSON types 
    to conform rigidly to the JSON specification 
    unless the database encoding is UTF8. 
    Attempts to directly include characters 
    that cannot be represented in the database encoding will fail; 
    conversely, 
    characters that can be represented in the database encoding 
    but not in UTF8 will be allowed.

    因为 PostgreSQL 只允许每个数据库使用一种字符集编码，
    所以如果数据库的编码不是 UTF8 ，
    那么 JSON 类型就不可能严格符合 JSON 规范的要求。
    尝试直接载入不能用数据库指定编码来表示的字符将会失败，
    与此相反，
    载入能够使用数据库编码来表示、但是不包含于 UTF8 的字符却是被允许的。

    RFC 7159 permits JSON strings to contain Unicode escape sequences denoted by \uXXXX. 
    In the input function for the json type, 
    Unicode escapes are allowed regardless of the database encoding, 
    and are checked only for syntactic correctness 
    (that is, that four hex digits follow \u). 
    However, 
    the input function for jsonb is stricter: 
    it disallows Unicode escapes for non-ASCII characters (those above U+007F) 
    unless the database encoding is UTF8. 
    It also insists that any use of Unicode surrogate pairs to designate characters outside the Unicode Basic Multilingual Plane be correct. 
    Valid Unicode escapes, except for \u0000, 
    are then converted to the equivalent ASCII or UTF8 character for storage.

    .. note::

        Note: Many of the JSON processing functions described in Section 9.15 will convert Unicode escapes to regular characters, and will therefore throw the same types of errors just described even if their input is of type json not jsonb. The fact that the json input function does not make these checks may be considered a historical artifact, although it does allow for simple storage (without processing) of JSON Unicode escapes in a non-UTF8 database encoding. In general, it is best to avoid mixing Unicode escapes in JSON with a non-UTF8 database encoding, if possible.

在将文本形式的 JSON 输入转换为 ``jsonb`` 数据的时候，
RFC 7159 中描述的基本类型会被高效地映射为 PostgreSQL 类型，
具体细节如表格 8-23 所示。

----

表格 8-23 JSON 基本类型以及相应的 PostgreSQL 类型

====================    =====================       ========================================================
JSON 基本类型           PostgreSQL 类型             注意事项
====================    =====================       ========================================================
``string``              ``text``                    参见之前对编码限制的讨论
``number``              ``numeric``                 不允许 ``NaN`` 值和 ``infinity`` 值
``boolean``             ``boolean``                 只接受小写的 ``true`` 和 ``false``
``null``                无                          SQL 的 ``NULL`` 和 JSON 的 ``null`` 是不同的概念
====================    =====================       ========================================================

----

因为类型转换的原因，
合法的 ``jsonb`` 数据具有少量 ``json`` 数据和抽象的 ``JSON`` 都不具有的额外限制，
这些限制和底层数据类型能够表示的数据有关。

具体来说，
``jsonb`` 会拒绝那些超出 PostgreSQL 数字类型范围的数字，
而 ``json`` 则不会这样做。
这种由实现定义的限制条件在 RFC 7159 中是被允许的。

然而在实际中，
因为很多其他实现都使用了 IEEE 754 双精度浮点数来表示 JSON 的数字基本类型（RFC 7159 已经预见了这一点并且遵循了这一规则），
所以这种问题出现的可能性并不低。
当 PostgreSQL 使用 JSON 格式与带有这种问题的系统进行信息交换的时候，
原本储存在 PostgreSQL 中的数据的数字精度可能会丢失。

另一方面，
正如表格所示，
某些 JSON 基本类型也是无法被转换为 PostgreSQL 类型的。


8.14.1. JSON 的输入语法和输出语法
--------------------------------------

JSON 数据类型的输入语法和输出语法都和 RFC 7159 中定义的一样。

以下展示的都是合法的 ``json`` 表达式或 ``jsonb`` 表达式：

::

    -- Simple scalar/primitive value
    -- Primitive values can be numbers, quoted strings, true, false, or null
    SELECT '5'::json;

    -- Array of zero or more elements (elements need not be of same type)
    SELECT '[1, 2, "foo", null]'::json;

    -- Object containing pairs of keys and values
    -- Note that object keys must always be quoted strings
    SELECT '{"bar": "baz", "balance": 7.77, "active": false}'::json;

    -- Arrays and objects can be nested arbitrarily
    SELECT '{"foo": [true, "bar"], "tags": {"a": 1, "b": null}}'::json;

和之前提到的一样，
在使用 ``JSON`` 值作为输入时，
并且不加任何额外处理进行打印的时候，
``json`` 会输出与输入完全相同的文本，
而 ``jsonb`` 则不会保留语义无关空白等细节。
下面的这个例子可以看出这种差别：

::

    SELECT '{"bar": "baz", "balance": 7.77, "active":false}'::json;
    json                       
    -------------------------------------------------
    {"bar": "baz", "balance": 7.77, "active":false}
    (1 row)

    SELECT '{"bar": "baz", "balance": 7.77, "active":false}'::jsonb;
    jsonb                       
    --------------------------------------------------
    {"bar": "baz", "active": false, "balance": 7.77}
    (1 row)    

另外一个不太重要的语义无关细节是，
``jsonb`` 中的数字会根据底层数字类型的行为来进行打印。
在实际中，
这意味着带有 E 符号的数字会被打印成无 E 符号的普通数字，
就像这样：

::

    SELECT '{"reading": 1.230e-5}'::json, '{"reading": 1.230e-5}'::jsonb;
            json           |          jsonb          
    -----------------------+-------------------------
    {"reading": 1.230e-5}  | {"reading": 0.00001230}
    (1 row)

正如上面的例子所示，
``jsonb`` 仍然会保留位于数字尾部的零，
尽管这些零对于相等判断这类操作来说是语义无关的。


8.14.2. 设计高效的 JSON 文档
---------------------------------

使用 JSON 来表示数据可以获得比使用传统的关系数据模型更大的灵活度，
这在需求不固定的环境里面非常具有吸引力。
这两种方法也可以在同一个应用程序里面同时存在并互相补充。
但是，
即使对于需要最大化灵活性的应用程序来说，
JSON 文档还是有比较固定的结构为好。
结构一般来说都是非强制的（尽管强制声明某些商业规则是有可能的），
但是拥有一个可预料的结构，
可以让那些针对表格里面多个文档的查询变得更容易编写。

JSON 数据和表格里面储存的任何其他数据类型一样，
都服从于相同的并发控制机制。
尽管储存一个巨大的文档是可行的，
但是因为每次更新都会请求一个针对整个行的行级锁（row-level lock），
所以请考虑将 JSON 文档限制在可控的大小之内，
以便减少更新事务带来的锁冲突。
在理想情况下，
每个 JSON 文档都应该被表示为一个原子数据（atomic datum） ——
这些数据可以独立被修改，
并且商业规则已经无法合理地将这些数据进一步分割为更小的数据。


8.14.3. ``jsonb`` 的包含检测操作符和存在检测操作符
------------------------------------------------------

``jsonb`` 类型的一个重要特性就是它可以检测一个 ``jsonb`` 值是否包含了另一个 ``jsonb`` 值，
而 ``json`` 类型并不具备这样的特性。

以下代码展示了一些例子：

::

    -- 一个简单的基本值只包含它自身
    SELECT '"foo"'::jsonb @> '"foo"'::jsonb;

    -- 左边的数组包含了右边的数组
    SELECT '[1, 2, 3]'::jsonb @> '[1, 3]'::jsonb;

    -- 数组元素的排列并不影响检测结果，所以以下检测的结果也为 true
    SELECT '[1, 2, 3]'::jsonb @> '[3, 1]'::jsonb;

    -- 同样地，重复的数组也不会影响检测结果
    SELECT '[1, 2, 3]'::jsonb @> '[1, 2, 2]'::jsonb;

    -- 左边的对象包含了右边的对象
    SELECT '{"product": "PostgreSQL", "version": 9.4, "jsonb":true}'::jsonb @> '{"version":9.4}'::jsonb;

    -- 在左边数组内部嵌套一个和右边数组相同的数组
    -- 并不代表边数组就包含了右边数组
    SELECT '[1, 2, [1, 3]]'::jsonb @> '[1, 3]'::jsonb;  -- 返回 false

    -- 但如果给右边数组加上一个嵌套层的话，那么包含关系就成立了
    SELECT '[1, 2, [1, 3]]'::jsonb @> '[[1, 3]]'::jsonb;

    -- 和前一个数组例子的情况类似，但这次被嵌套的是键值对
    SELECT '{"foo": {"bar": "baz"}}'::jsonb @> '{"bar": "baz"}'::jsonb;  -- 返回 false

包含检测的一般原则是，
被包含的对象必须在结构和数据内容上与实施包含的对象一致，
而实施包含的对象可能是在丢弃了某些非匹配数组元素或非匹配对象键值对之后才达成这种一致。
需要记住的一点是数组元素的摆放顺序并不会影响检测的结果，
并且为了提高效率，
重复的数组元素只会被检测一次。
作为“结构必须匹配”这个一般原则的一个特殊例外，
一个数组可以包含一个基本值：

::

    -- 这个数组包含了基本字符串值：
    SELECT '["foo", "bar"]'::jsonb @> '"bar"'::jsonb;

    -- 这个例外并不是可互换的 —— 比如在这个例子里面，包含条件就不成立：
    SELECT '"bar"'::jsonb @> '["bar"]'::jsonb;  -- 返回 false

除了包含检测操作符之外，
``jsonb`` 还支持存在检测操作符，
这两者之间只有外型上的不同：
这种操作符会检测给定的字符串（一个 ``text`` 值）是否为对象的键或数组的顶层元素。
除了有特别注明的例子之外，
下面展示的例子都返回 ``true`` ：

::

    -- 字符串是否为数组元素？
    SELECT '["foo", "bar", "baz"]'::jsonb ? 'bar';

    -- 字符串是否为对象的键？
    SELECT '{"foo": "bar"}'::jsonb ? 'foo';

    -- 操作在进行检测时不会考虑对象的值：
    SELECT '{"foo": "bar"}'::jsonb ? 'bar';  -- 返回 false

    -- 和包含检测一样，存在检测只会在顶层进行：
    SELECT '{"foo": {"bar": "baz"}}'::jsonb ? 'bar'; -- 返回 false

    -- 对于相同的字符串，存在检测操作符会返回 true
    SELECT '"foo"'::jsonb ? 'foo';

JSON 对象比数组更适合用在需要对多个键或者多个元素进行包含检测或存在检测的场景，
因为 JSON 对象和数组不同，
它在内部对搜索进行了优化，
并且不需要进行线性查找。

`9.15 节 <http://www.postgresql.org/docs/9.4/interactive/functions-json.html>`_\ 列出了包括包含检测操作符和存在检测操作符在内的所有 JSON 操作符和函数。


8.14.4 ``jsonb`` 索引
------------------------

GIN 索引可以提升在多个 ``jsonb`` 文档里面查找特定键或者特定键值对的效率。
PostgreSQL 提供了两种 GIN 操作符类（operator class），
它们在性能和灵活性上有不同的取舍。

``jsonb`` 的默认 GIN 操作符类 ``jsonb_ops`` 支持为带有 ``@>`` 、 ``?`` 、 ``?&`` 、 ``?|`` 操作符的查询创建索引（\ `表格 9-41 <http://www.postgresql.org/docs/9.4/interactive/functions-json.html#FUNCTIONS-JSONB-OP-TABLE>`_\ 记录了这些操作符的详细语义）。
以下是一个创建 ``jsonb_ops`` 索引的例子：

::

    CREATE INDEX idxgin ON api USING gin (jdoc);

非默认的 GIN 操作符类 ``jsonb_path_ops`` 只支持为带有 ``@>`` 操作符的查询创建索引。
以下是一个创建 ``jsonb_path_ops`` 索引的例子：

::

    CREATE INDEX idxginp ON api USING gin (jdoc jsonb_path_ops);

假设有一个表，
它使用文档来储存第三方 web 服务传来的 JSON 数据，
以下是这中 JSON 数据的一个例子：

::

    {
        "guid": "9c36adc1-7fb5-4d5b-83b4-90356a46061a",
        "name": "Angela Barton",
        "is_active": true,
        "company": "Magnafone",
        "address": "178 Howard Place, Gulf, Washington, 702",
        "registered": "2009-11-07T08:53:22 +08:00",
        "latitude": 19.793713,
        "longitude": 86.513373,
        "tags": [
            "enim",
            "aliquip",
            "qui"
        ]
    }

如果我们将这个文档储存到 ``api`` 表的 ``jdoc`` 列里面，
并且为 ``jdoc`` 列创建 GIN 索引，
那么下面的这个查询就会用到我们创建的索引：

::

    -- 查找键 company 的值为 Magnafone 的文档
    SELECT jdoc->'guid', jdoc->'name' FROM api WHERE jdoc @> '{"company": "Magnafone"}';

与此相反，
因为下面这个查询并非直接针对带有索引的 ``jdoc`` 列，
所以尽管查询用到了可索引的 ``?`` 操作符，
但这个查询还是没办法用到 ``jdoc`` 行的索引：

::

    -- 查找那些在键 tags 里面包含了数组元素 qui 的文档
    SELECT jdoc->'guid', jdoc->'name' FROM api WHERE jdoc -> 'tags' ? 'qui';

尽管只要恰当地使用表达式索引，
就可以在上面的查询中用到索引，
但如果我们经常需要在 ``tags`` 数组里面查找特定元素的话，
那么最好的做法还是直接为 ``tags`` 数组创建一个索引，
就像这样：

::

    CREATE INDEX idxgintags ON api USING gin ((jdoc -> 'tags'));

现在，
``WHERE jdoc -> 'tags' ? 'qui'`` 将被识别为可索引操作符 ``?`` 对已索引表达式 ``jdoc -> 'tags'`` 的应用。
（\ `11.7 节 <http://www.postgresql.org/docs/9.4/interactive/indexes-expressional.html>`_\ 介绍了表达式索引的更多相关信息。）

另一种进行查询的方法是使用包含检测操作符，
以下是一个例子：

::

    -- 查找那些在 tags 键里面包含了数组元素 qui 的文档
    SELECT jdoc->'guid', jdoc->'name' FROM api WHERE jdoc @> '{"tags": ["qui"]}';

针对 ``jdoc`` 列的 GIN 索引支持这个查询。
不过请记住，
GIN 索引会储存 ``jdoc`` 列所有键和值的副本，
而之前展示的表达式索引只会储存 ``tags`` 键的值。

因为 GIN 索引支持针对任何键的查询，
所以它具有更大的灵活性，
但定向的表达式索引可以节约更多空间，
并且搜索的速度有时候甚至比 GIN 索引还要快。

尽管 ``jsonb_path_ops`` 索引只支持带有 ``@>`` 操作符的查询，
但是与默认的 ``jsonb_ops`` 索引相比，
``jsonb_path_ops`` 索引有明显的性能优势。
对于相同的数据，
建立 ``jsonb_path_ops`` 索引所需的空间比建立 ``jsonb_ops`` 索引所需的空间要少得多，
并且 ``jsonb_path_ops`` 索引在处理某些特定的查询时拥有更好的性能，
当查询包含的键频繁地在数据中出现时，
更是如此。

``jsonb_ops`` 和 ``jsonb_path_ops`` GIN 索引之间的技术区别在于，
前者会为数据中的每个键和值都创建独立的索引项，
而后者只会为数据中的每个值创建索引项。

- 每个 ``jsonb_path_ops`` 索引项基本上就是一个由被索引的键和值组成的散列。
  举个例子，
  为 ``{"foo": {"bar": "baz"}}`` 建立 ``jsonb_path_ops`` 索引将产生一个新的索引项，
  这个索引项是一个由 ``foo`` 、 ``bar`` 、 ``baz`` 三个值组成的散列，
  因此针对这个结构的包含检测查询最终将会产生一次非常明确的索引搜索，
  但是这种实现也使得 PostgreSQL 没办法判断 ``foo`` 到底是被索引的键还是被索引的值。

- 与此相反，
  ``jsonb_ops`` 索引会为分别为 ``foo`` 、 ``bar`` 、 ``baz`` 各创建一个索引，
  在进行包含检测查询的时候，
  PostgreSQL 就会在包含了这三个索引项的行里面进行查找。

尽管 ``jsonb_ops`` 索引可以比较高效地执行像 ``AND`` 这样的搜索，
但是比起等价的 ``jsonb_path_ops`` 搜索，
``jsonb_ops`` 索引在明确性和速度上还是要略逊一筹，
当 ``jsonb_ops`` 创建了非常多的索引行，
而 PostgreSQL 只需要从这些行里面搜索少量几个被索引项的时候，
更是如此。

``jsonb_path_ops`` 索引的缺点在于，
它不会为类似 ``{"a": {}}`` 这样不包含任何值的 JSON 结构创建索引，
因此在对包含这样结构的文档进行搜索时，
就需要耗费大量的时间对整个索引进行搜索。
因此，
``jsonb_path_ops`` 并不适用于需要经常执行这种查询的应用程序。

``jsonb`` 还支持 B 树索引和散列索引，
但是只有当“检查整个 JSON 文档是否相等”这个操作非常重要的时候，
人们才会使用这两种索引。
很少会有人对 B 树如何排列 ``jsonb`` 数据感兴趣，
但为了完整性考虑，
这里还是给出详细的 B 树对比规则：

::

    Object > Array > Boolean > Number > String > Null

    Object with n pairs > object with n - 1 pairs

    Array with n elements > array with n - 1 elements

键值对数量相同的对象会按照以下顺序来进行对比：

::

    key-1, value-1, key-2 ...

注意对象的键是按照储存顺序来进行对比的，
在实际中，
因为较短的键会储存在较长的键的前面，
所以这可能会导致一些不符合直觉的对比结果，
比如说：

::

    { "aa": 1, "c": 1} > {"b": 1, "d": 1}

类似的，
元素数量相同的数组会按照以下顺序来进行对比：

::

    element-1, element-2 ...

基本 JSON 值进行对比时，
使用和底层 PostgreSQL 数据类型一样的对比规则。
而字符串则根据数据库的默认编码来进行对比。
