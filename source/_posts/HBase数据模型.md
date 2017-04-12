在 HBase 中，数据存储在表中，其中具有行和列。这个术语与关系型数据库（RDBMS）的重叠，但这不是有用的类比。相反，将 HBase 的表想象为多维的映射可能更有帮助。

### *HBase 数据模型术语*
**表**
HBase 表由多个行组成。

**行**
HBase 中的行由行键及一个或多个关联了值的列组成。行按照存储时的行键的字母序排序。因此，行键的设计非常重要。目标是以一种将相关的行存储在彼此相近的位置的方式存储。一种常用的行键模式是网站的域名。如果你的行键是域名，你可能应该以逆序存储它们（org.apache.www，org.apache.mail，org.apache.jira）。这种方式下，所有的 Apache 域名在表中是彼此接近的，而不是基于子域名的第一个字母而彼此分离。

***********************************************
**HBase 中，一个行键只能标识一个行，还是可以标识多个行？**
***********************************************

**列**
HBase 中的列由列族（column family）和列修饰符组成，它们由 `:` (冒号)
字符分割。

**列族（column family）**
列族物理上共同定位一组列及其值，通常出于性能原因。每个列族具有一系列存储属性，比如它的值是否应该缓存在内存中，它的数据如何压缩或它的行键如何编码，及其它。表中的每一行具有相同的列族，尽管给定的行可能不存储任何给定的列族。

***********************************************
**表中的一行具有相同的列族，也就是说，每一行最多只能有一个列族。由此为何不直接将行键当作列族来用？在 HBase 中列族与行键之间是否有关系？**
***********************************************

**列修饰符**
列修饰符被添加到列族以提供对给定的一片数据的索引。给定列族 `content`，一个列修饰符可能是 `content:html`，而另一个可能是 `content:pdf`。尽管列族在创建表时就固定了下来，而列修饰符是可变的，且在行之间可能存在这巨大的差异。

**单元（Cell）**
单元是行，列族和列修饰符的结合，且包含了值和时间戳，其表示值的版本。

***********************************************
**定位一个单元的过程为：行键 -> 列族 -> 列修饰符？**
***********************************************

**时间戳**
时间戳伴随每个值写入，它是值的给定版本的标识符。默认情况下，时间戳表示 RegionServer 上数据写入时的时间，但你在将数据放入单元时，可以指定一个不同的时间戳值。

## 概念视图
你可以在 Jim R. Wilson 的博客文章 [理解 HBase  和 BigTable](https://dzone.com/articles/understanding-hbase-and-bigtab) 中读到非常易于理解的关于 HBase 数据模型的解释。另外一份很好的解释是 Amandeep Khurana 的 PDF 文档 [Introduction to Basic Schema Design](http://0b4af6cdc2f0c5998459-c0245c5c937c5dedcca3f1764ecc9b2f.r43.cf2.rackcdn.com/9353-login1210_khurana.pdf) 

阅读不同的视角可能有助于对 HBase schema 设计有一个深入的理解。链接的文章涵盖与本节中的信息相同的基础。

下面的例子是 [BigTable](http://research.google.com/archive/bigtable.html) 论文的第 2 页中的例子经过微小的改动而来。有一个称为 `webtable` 的表，其中包含了两行（`com.cnn.www` 和 `com.example.www`）及三个名为 `contents`，`anchor` 和 `people` 的列族。在这个例子里，第一行 (`com.cnn.www`) 中，`anchor` 包含两列 (`anchor:cssnsi.com`, `anchor:my.look.ca`)，`contents` 包含一列 (`contents:html`)。这个例子中，包含以 `com.cnn.www` 为行键的 5 个版本的行，及以 `com.example.www` 为行键的一个版本的行。`contents:html` 列修饰符包含给定网站的完整 HTML。每个 `anchor` 列族的修饰符包含外部站点，其链接到由行表示的网站，以及其在其链接的 anchor 中使用的文本。`people` 列族表示与站点关联的人。

**列名**
按惯例，列名由它的列族前缀和一个 *修饰符* 组成。比如，列 `contents:html` 由列族 `contents` 和修饰符 `html`。冒号字符（`:`）分割列族和列族 *修饰符*。

表 4. `webtable`

Row Key | Time Stamp | ColumnFamily `contents` | ColumnFamily `anchor `| ColumnFamily `people`
-------|----------|----------|----------|----------
"com.cnn.www" | 	t9 | 	 | 	anchor:cnnsi.com = "CNN"| 	
"com.cnn.www" | 	t8 | 	 | 	anchor:cnnsi.com = "CNN"| 	
"com.cnn.www" | 	t6 |  contents:html = "<html>…​"	 | 	| 	
"com.cnn.www" | 	t5 | contents:html = "<html>…​"	 | 	| 	
"com.cnn.www" | 	t3 | contents:html = "<html>…​"	 | 	| 	

这个表中看上去为空的单元格不占用空间，或者实际上不存在，在 HBase 中。这就是HBase“稀疏”的原因。表的视角不是惟一可能的看待 HBase 中的数据的方式，或者甚至不是最精确的。下面以多维映射表示了相同的信息。这只是一个以说明为目的的模型，可能不是非常的精确。
```
{
  "com.cnn.www": {
    contents: {
      t6: contents:html: "<html>..."
      t5: contents:html: "<html>..."
      t3: contents:html: "<html>..."
    }
    anchor: {
      t9: anchor:cnnsi.com = "CNN"
      t8: anchor:my.look.ca = "CNN.com"
    }
    people: {}
  }
  "com.example.www": {
    contents: {
      t5: contents:html: "<html>..."
    }
    anchor: {}
    people: {
      t5: people:author: "John Doe"
    }
  }
}
```

## 物理视图
尽管在概念层表可以被视为一个行的稀疏集合，但在物理上它们是根据列族存储的。新的列修饰符 (column_family:column_qualifier) 可以在任何时间被添加到现有的列族。

表 5. 列族 `anchor`

Row Key | Time Stamp | ColumnFamily `anchor `|
-------|----------|----------|
"com.cnn.www" | 	t9 | anchor:cnnsi.com = "CNN"  |
"com.cnn.www" | 	t8 | anchor:my.look.ca = "CNN.com"  |

表 5. 列族 `contents`

Row Key | Time Stamp | ColumnFamily `contents: `|
-------|----------|----------|
"com.cnn.www" | 	t6 | contents:html = "<html>…​"  |
"com.cnn.www" | 	t5 | contents:html = "<html>…​"  |
"com.cnn.www" | 	t3 | contents:html = "<html>…​"  |

在概念视图中显示的空单元不再存储。这样对时间戳 `t8` 请求 `contents:html` 列的值将不返回值。类似地，对时间戳 `t9` 请求 `anchor:my.look.ca` 列的值也将不返回值。然而，如果没有提供时间戳，将返回特定列最近的值。给定多个版本，最近的也是第一个被发现的，由于时间戳是降序存储的。这样如果没有指定时间戳，请求行 `com.cnn.www` 中的所有列的值将返回：时间戳 `t6` 的 `contents:html` 的值，时间戳 `t9` 的 `anchor:cnnsi.com` 的值，及时间戳 `t8` 的 `anchor:my.look.ca` 的值。

更多关于 Apache HBase 内部如何存储数据的信息，请参考 [regions.arch](http://hbase.apache.org/book.html#regions.arch)。

## 命令空间
命名空间是与关系型数据库系统中的数据库类似的表的逻辑分组。这种抽象为即将到来的多租户相关特性奠定了基础：

* 配额管理 ( [HBASE-8410](https://issues.apache.org/jira/browse/HBASE-8410) )  - 限制一个命名空间可消费的资源（ 即区域，表）的数量。
 * 命名空间安全管理 ( [HBASE-9206](https://issues.apache.org/jira/browse/HBASE-9206) ) - 为租户提供另一层的安全管理。
  * 区域服务器组 ( [HBASE-6721](https://issues.apache.org/jira/browse/HBASE-6721) ) - 命名空间/表可以被固定在 RegionServers 的子集上以保证一个过程级别的隔离。
  
### 命名空间管理
命名空间可以被创建、移除或者改变。在表格创建期间通过指定表单的完全限定表名来确定命名空间成员资格：
```
<table namespace>:<table qualifier>
```

示例. 例子
```
#Create a namespace
create_namespace 'my_ns'
```

```
#create my_table in my_ns namespace
create 'my_ns:my_table', 'fam'
```

```
#drop namespace
drop_namespace 'my_ns'
```

```
#alter namespace
alter_namespace 'my_ns', {METHOD => 'set', 'PROPERTY_NAME' => 'PROPERTY_VALUE'}
```

### 预定义命名空间

有两个预定义的特殊的命名空间：

 * hbase - 系统命名空间，用于包含 HBase 内部的表。
 * default - 没有显式地指定命名空间的表将自动地落入这个命名空间。
 
 示例. 例子
 
```
#namespace=foo and table qualifier=bar
create 'foo:bar', 'fam'
```

```
#namespace=default and table qualifier=bar
create 'bar', 'fam'
```
 
## 表
表在模式定义时间前面声明。

## 行
行键是未解释的字节。行按字典顺序排序，最低顺序首先出现在表中。空字节数组用于表示表的命名空间的开始和结束。

## 列族
Apache HBase中的列被分组为列族。一个列族所有的列成员具有相同的前缀。比如，列 *courses:history* 和 *courses:math* 都是 *courses* 列族的成员。冒号（`:`）将列族与列族限定符分割开。列族前缀必须由 *可打印* 字符组成。合格的尾部，列族修饰符，可以由任意字节组成。列族必须在模式定义时声明，而列不需要在模式时定义，但可以在表启动和运行时动态处理。

物理上而言，在文件系统上所有的列族成员被存储在一起。由于调整和存储规范是在列族级别完成的，因此建议所有列族成员具有相同的通用访问模式和大小特性。

## 单元
*{row, column, version}* 三元组精确指定 HBase 中的一个 `单元`。单元内容是未解释的字节。

## 数据模型操作
四个主要的数据模型操作是 Get，Put，Scan 和 Delete。操作通过 [表](http://hbase.apache.org/apidocs/org/apache/hadoop/hbase/client/Table.html) 实例应用。

### Get
[Get](http://hbase.apache.org/apidocs/org/apache/hadoop/hbase/client/Get.html) 返回特定行的属性。Get 通过 [Table.get](http://hbase.apache.org/apidocs/org/apache/hadoop/hbase/client/Table.html#get(org.apache.hadoop.hbase.client.Get)) 执行。

### Put
[Put](http://hbase.apache.org/apidocs/org/apache/hadoop/hbase/client/Put.html) 向表中添加新行（如果 key 是新的）或可以更新已有的行（如果 key 已经存在）。Puts 通过 [Table.put](http://hbase.apache.org/apidocs/org/apache/hadoop/hbase/client/Table.html#put(org.apache.hadoop.hbase.client.Put)) (writeBuffer) 或 [Table.batch](http://hbase.apache.org/apidocs/org/apache/hadoop/hbase/client/Table.html#batch(java.util.List,%20java.lang.Object%5B%5D)) ( 非writeBuffer )执行。

### Scans
[Scan](http://hbase.apache.org/apidocs/org/apache/hadoop/hbase/client/Scan.html) 允许对多个行的指定属性进行迭代。假设表中填充有具有键“row1”，“row2”，“row3”的行，然后是另一组具有键“abc1”，“abc2”和“abc3”的行。下面的例子展示了如何设置一个 Scan 实例来返回以 "row" 开始的行。

```
public static final byte[] CF = "cf".getBytes();
public static final byte[] ATTR = "attr".getBytes();
...

Table table = ...      // instantiate a Table instance

Scan scan = new Scan();
scan.addColumn(CF, ATTR);
scan.setRowPrefixFilter(Bytes.toBytes("row"));
ResultScanner rs = table.getScanner(scan);
try {
  for (Result r = rs.next(); r != null; r = rs.next()) {
    // process result...
  }
} finally {
  rs.close();  // always close the ResultScanner!
}
```

注意，通常为 scan 指定特定的停止点的最简单的方式是使用 [InclusiveStopFilter](http://hbase.apache.org/apidocs/org/apache/hadoop/hbase/filter/InclusiveStopFilter.html) 类。

### Delete
[Delete](http://hbase.apache.org/apidocs/org/apache/hadoop/hbase/client/Delete.html) 从表中移除一行。删除通过 [Table.delete](http://hbase.apache.org/apidocs/org/apache/hadoop/hbase/client/Table.html#delete(org.apache.hadoop.hbase.client.Delete)) 执行。

## 版本

## 排序

## 列元数据

## Joins

## ACID
参考 [ACID 语义](http://hbase.apache.org/acid-semantics.html)。Lars Hofhansl 也写了一篇关于 [HBase 中的 ACID](http://hadoop-hbase.blogspot.com/2012/03/acid-in-hbase.html) 的笔记。







[原文](http://hbase.apache.org/book.html#datamodel)
