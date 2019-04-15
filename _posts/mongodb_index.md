---
title: MongoDB 索引创建参数partialFilterExpression高级用法
date: 2017-11-05 18:50:15
tags: mongodb index
---

### 业务场景

前段时间，因业务需求，对原来MongoDB数据库表A中的a, b, c三个字段的普通索引改为做唯一性存储，但不将空值(null)和空字符串("")计入唯一性的限制内。

### 解决方案

#### 唯一索引

创建唯一索引的方法很简单:

```
db.A.createIndex({a: 1}, {unique: true, background: true});
db.A.createIndex({b: 1}, {unique: true, background: true});
db.A.createIndex({c: 1}, {unique: true, background: true});
```
其中`unique`保证的字段的唯一限制，`background: true`在创建索引的时候不会阻塞数据库服务。

但是怎么将空值和空字符串给过滤掉呢？

#### 特殊唯一索引

笔者首先想到的是createIndex方法里面的sparse参数，但官方文档说明:

> Sparse indexes only contain entries for documents that have the indexed field。

意思就是只对包含有该索引字段(field)的文档做索引存储。该方案明显不符合业务需求。

笔者再仔细翻阅文档，发现创建索引方法有参数`partialFilterExpression`的介绍:

> If specified, the index only references documents that match the filter expression.

据说明，可以知道我们可以通过这个参数来筛选特殊的文档做索引。

那么接下来就可以这样做了，

```
db.A.createIndex({a: 1}, {unique: true, background: true, partialFilterExpression: {a: {$type: "string", $gt: ""}}});
db.A.createIndex({b: 1}, {unique: true, background: true, partialFilterExpression: {b: {$type: "string", $gt: ""}}});
db.A.createIndex({c: 1}, {unique: true, background: true, partialFilterExpression: {c: {$type: "string", $gt: ""}}});
```
参数`partialFilterExpression: {a: {$type: "string", $gt: ""}}`将空值和空字符串成功过滤掉。开心...

那么这样就成功完成了需求了吗？

No, too young too simple!

还有一个很严重的问题.官网文档这样说:

> To use the partial index, a query must contain the filter expression (or a modified filter expression that specifies a subset of the filter expression) as part of its query condition.

意思是即使这样创建了索引，但是你的普通查询是不能正常使用这个索引的。

Oh my god! 创建了索引没法正常使用，那要它来干嘛？

别灰心，这里肯定是有用的，至少它能保证存储数据的唯一性。

#### 最终解决方案

接下来，我只需要创建一个普通索引，并且覆盖该字段，即可在查询的时候也能正常命中索引了。完整代码如下:
```
db.A.createIndex({a: 1}, {background: true});
db.A.createIndex({b: 1}, {background: true});
db.A.createIndex({c: 1}, {background: true});
db.A.createIndex({a: 1, another: 1}, {unique: true, background: true, partialFilterExpression: {a: {$type: "string", $gt: ""}}});
db.A.createIndex({b: 1, another: 1}, {unique: true, background: true, partialFilterExpression: {b: {$type: "string", $gt: ""}}});
db.A.createIndex({c: 1, another: 1}, {unique: true, background: true, partialFilterExpression: {c: {$type: "string", $gt: ""}}});
```
各位好！解释一下，前三行代码保证了字段a, b, c能在查询的时候命中索引。

后三行代码中的another字段其实在数据库是不存在的，因为a, b, c字段在数据库中已经创建了索引，mongodb不允许对完全一样的字段重复创建索引，即使参数不一样。那么就可以这样添加一个不存在的字段，既节省内存，又能解决查询问题。

<b> *Done!* 现在的解决方案已经在拥有上千万数据的MongoDB集群上平稳运行了。</b>

#### 总结

**缺点**: *由于一个字段创建了两个索引，会占用更多的内存。本项目中大约多占用了400M内存，在可接受的范围内。*

**优点**: *只需添加三个特殊索引，无需修改原来的索引创建，同时业务代码修改较小。对于线上的数据库影响较小，且能满足业务需求。*


** *申明：该网站所有文章均为原创，转载请著名出处:`http://blog.lican.site`，谢谢！* **

<div id="SOHUCS" sid="mongodb index"></div>
