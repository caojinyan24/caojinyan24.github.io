---
layout: post
title:  "mysql之text"
date:   2019-05-30 23:02:24 +0800
categories: 基础
tags: mysql
---

今天组内一个同学创建mysql数据表,聊起一个字段比较大,要不要设成text.印象里记得text类型会存储到另一个地方,不限制大小.刚回来路上百度了下,看到的都是`text跟varchar最大存储空间一样,最好是用varchar,不要用text的言论`.换google,找到一个比较官方的说法[mysql官方文档](https://dev.mysql.com/doc/refman/8.0/en/blob.html)

总结几个关键点:

1. 首先BLOB和TEXT类型也是有最大存储长度限制的,各个mysql类型的最大长度如下
![](/_pic/201903/201905/datatype.png)
2. 当使用严格的sql模式时,如果存储text或blob类型的数据超出了最大长度限制,会发送错误.而varchar类型发生截断时只是发生警告.
3. blob和text列会放在磁盘存储,因此在查询时会导致性能问题.所以尽量避免查询结果中包含blob和text类型的字段.
4. 为text或blob添加索引时,只能设置前缀字符为索引.所以需要指定前缀的长度. 如果TEXT列被索引，则索引条目比较在末尾以空格填充。这意味着，如果索引需要唯一值，则对于仅在尾随空格数方面不同的值，将出现重复键错误。例如，如果表包含'a'，则尝试存储'a '会导致重复键错误。BLOB列则不会.

随便百度出来的人云亦云的答案不靠谱,有问题还是要自己去看官方文档