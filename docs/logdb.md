日志分析服务（LogDB）提供针对日志类数据的存储、检索与分析服务，用户无需开发就能快捷完成数据定制化分词、存储、检索、分析功能，帮助提升运维、运营效率，快速查找和定位问题，高效索引和搜索海量数据。日志分析服务提供多种用户场景的分析需求，包括日志高速索引、查询等实时场景的深度性能优化，也包括全文检索、文档更新等永久存储场景的高性能优化。

![](http://docs.qiniucdn.com/logdb.png)

## 概念&术语

```
repo      fields      analyzer      retention     
```

</br>

现实中，我们的日志是多样的，在概念和术语的解释中，我们将尽量使用具有代表性的真实日志来解释。

### 日志仓库（repo）

类似于关系型数据库的`表（table）`概念，在日志分析服务中，我们叫它日志仓库。

### 日志字段（fields）

像关系型数据库一样，fields的概念和columns一样，但在日志分析服务中，fields支持的类型有以下几种：

|类型|解释|数据样例|
|:--|:--|:--|
|long|64位整数|998|
|float|单精度64位浮点|10.24|
|date|日期类型，默认格式为RFC3339，可自定义格式|`2017-01-01T15:00:25Z07:00`|
|string|字符串类型，可设置分词器和主键|"qiniu.com"|
|boolean|布尔类型，值为true或false|false|
|object|json格式|-|
|ip|ip地址|127.0.0.1|
|geo_point|经纬度坐标|[ -71.34, 41.12 ]|

### 分词方式（analyzer）

从海里的日志中过滤出我们需要的内容，需要精确的搜索条件，为了确保我们输入的条件能够和日志进行精确匹配，我们需要选择合适的分词方式，目前日志分析服务提供以下几种分词方式：

**不分词**

不分词的含义其实是“keyword分词”，意思就是，你必须完整的输入某一个field的内容，才能被搜索到，我们举个例子来说明：

```
假设目前有1个field为A，它的内容是"abcdefg"
假设我们选择了不分词方式，那么如果想搜索到这一条内容，需要在条件框输入：
A:"abcdefg"
这样才能搜索到，如果我们按照以下几种方式，则无法搜索到：
A:"a"
A:"abc"
A:"edf"
```

**不分词不索引**

不分词不索引的含义是指：在任何情况下，输入任何条件，都不会搜索到该field的内容，但在搜索其他field时，如果是整条搜索，那么这个field的内容也会显示出来。

```
假设现在有2个field，分别为：
A:"abcdefg",B:"12345678"
无论我们在条件框输入有关B的任何条件，都不会搜索到
```

**标准分词**

以unicode字符作为结束的标识，过滤掉大部分标点符号，将所有字母变为小写，搜索时只要符合搜索条件的词语出现，就会被搜索到。

```
假设现在有1个field为A，它的内容是"linux-chrome,what's your&name?"
那么使用标准分词，将会被分为："linux,chrome,what's,your,name"
我们的搜索条件包含这些词时，会被搜索到
```

!> 注意：标准分词只适用于全英文字符的field。

**空白分词**

以单词头部和尾部的空格作为分割条件，输入两个空格内的完整内容，即可搜索到相应内容。

```
例：
A="张三 李四 王五"

需要输入以下条件，均可搜到内容：
A:"张三"
A:"李四"
A:"王五"

如果输入以下条件，则无法搜索到内容：
A:"张"
A:"张三 李四"
```

**path分词**

以linux系统路径来进行分词，当搜索时，可以输入完整的路径或者路径前缀来进行匹配，如果输入的条件不是一个正确的前缀，那么将无法正确呈现日志内容。

```
例：
filed A="/usr/local/action.log"

那么需要输入以下条件，均可搜到内容：
A:"/usr"
A:"/usr/local"
A:"/usr/local/action.log"

如果输入以下条件，则无法搜索到内容：
A:"/u"
A:"usr"
A:"/action.log"
```


### 存储时限（retention）

创建仓库时，我们需要指定这个属性，它的意思是指这个仓库内的每一条日志都会被存储和retention一致的天数，超过这个时间的数据会被自动删除，当retention指定为0时，表示永久存储。


## 搜索范式

日志分析服务提供Lucene、SQL共2种语法来进行日志搜索，但需要注意，一旦有任何数据包含以下符号，无论使用哪种语法，在搜索时，需要以`双引号（""）`包含起来：

```
+  -  
&&  ||  !
( )  { }  [ ] 
^  ”  ~  *  ?  :  \
```


### 使用 lucene 语法搜索

**条件编写规范**

|名称|语义|
|:--|:--|
|*|查询所有内容|
|AND|query1 AND query2，查询交集|
|OR	|query1 OR query2，查询并集|
|NOT|query1 AND NOT query2，表示符合query1，不符合query2的结果|
|()	|把一个或多个query合并成一个query，提升优先级|
|[]	|区间查询，包括边界|
|{}|区间查询，不包括边界|
|\|转义字符|
|>,=,<,<=,>=|区间查询|

**正则表达式查询**

正则表达式查询条件编写时，以`"/"`开头和结尾。

比如：`name:/joh?n(ath[oa]n)/`

**通配符查询**

使用`?`代替一个字符，`*`代替0或者多个字符

比如：`qu?ck bro*`

* 注意使用这个查询会消耗大量资源，并且速度会降低。


**查询举例：**

字段名称`name`，类型`string`，包含内容`a`的记录：

?>name:a

字段名称`ip`，类型`string`，包含内容`a`或`b`的记录：

?>ip:a OR ip:b

字段名称`hosts`，类型`string`，包含内容`a`或者`b`,不包含`c`的记录：

?>(hosts:a OR hosts:b) AND (NOT hosts:c)

字段名称`ip`，类型`string`，包含内容`a`或`b`，同时字段名称`hosts`，类型`string`，包含内容`c`的记录：

?>(ip:a OR ip:b) AND (hosts:c)

字段名称`createTime`，类型`date`，包含内容`2016-1-1`到`2016-1-2`的记录：

?>createTime:[2016-01-01 TO 2016-01-02]

字段名称`count`，类型`long`或者`float`，内容大于5的记录： 

?>count:>5

更多高级语法和使用方式请参考Lucene Query。


### 使用 sql 搜索（即将开放）