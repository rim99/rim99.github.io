---
layout: post
title: "Data Vault 2.0的结构简介"
date: 2019-08-20 11:00:34 +0800
categories: 原创
---

> 本文为[《Building a Scalable Data Warehouse with Data Vault 2.0》](https://www.oreilly.com/library/view/building-a-scalable/9780128026489/)的阅读笔记

## 合格的企业数据仓库

合格的企业数据仓库（Enterprise Data Warehouse，EDW）应当具备以下特点：
* Access：快速连接，快速处理请求，请求结果易于理解
* 多主题：数据仓库能够面向多个业务部门，保证同时满足不同部门的业务要求
* 只有一个真相：不同部门的请求结果应保证一致性
* 保证数据质量
* 可伸缩性：能够应对不断增长的数据量和用户量
* Big Data的特点
  - volume：超过硬件处理能力的数据量
  - velocity：需要快速加载外部数据源
  - variety：各种非结构化的数据，文本音频图片等等
* 高性能：主要体现在处理请求和加载数据两个方面

为了应对快速变化的业务需求，数据仓库必须保证足够的灵活性。Data Vault 2.0 便是一种灵活的建模方案。

## Data Vault 2.0 的分层结构

按照从源头到终端的数据流动顺序，Data Vault 2.0 包含以下几层：
* Staging Area layer，强制实现
* Data Warehouse layer，强制实现
* Business Vault layer，可选实现
* Information Delivery/Mart layer，强制实现

### 数据的转换

当数据从源头进入EDW，以及从EDW呈现给终端用户时，会经历数据转换的过程，这个转换依赖于两类规则：
* Hard 业务规则：这类规则不会修改数据内涵，会修改其存储形式，依照业务本身定义。规则相对稳定。
* Soft 业务规则：由业务人员确定，一般会频繁的变化。

过去，这两类规则都体现在数据从源头导入EDW的阶段。然而灵活多变的Soft规则导致ETL工具不断的重构，降低了EDW系统的伸缩性。
Data Vault 2.0则将ETL拆分为两个部分。数据从源头进入EDW时执行Hard规则，数据将要呈现给终端用户的时候执行Soft规则。减少了ETL工具的重构开销。

### Staging Area layer

这一层主要作用是快速的从操作性数据库（operational database）提取数据，减少对业务系统的影响。

不包含历史数据，只存储下一次需要加载到DW层的数据；如果又一次ETL执行失败，需要清除加载到一半的数据，重新执行ETL。

staging通常需要完整的复制source表中的数据，需要包含主键但不包含外键和索引之类确保数据引用完整性的数据。

staging环境的表中需要包含如下列作为元数据，会将来加载到DW层：
* 顺序编号：记录数据在source系统的顺序
* 时间戳：记录数据进入edw的时间
* 记录源
* business key的hash计算与其组合的hash计算：用于标示数据身份

这一层需要快速写入数据，容错要求不高，因此推荐使用RAID0磁盘架构。如果数据完全结构化且类型明确，可以使用关系型数据库存储数据，否则可使用NoSQL。

### Data Warehouse layer

又称作Raw Data Vault layer。

这一层是主要的数据存储层，存储所有随时间变化的历史数据。这里的数据都是面向功能的（function-oriented）。

这一层存储了所有历史性的数据，难以恢复，因此要求系统有良好的纠错能力，建议使用RAID1或RAID5磁盘架构。同时建议使用异地机房确保数据的安全。机房间通信安全可以考虑用SSL或VPN来保证。

这一层使用Hub、Link、Satellite三种表来组织数据。

* Hub 存储业务实体的元信息metadata，例如业务键 business key，一个业务键对应一个业务实体 business object
* Satellite 包含了Hub和Link对应的业务context
* Link 链接各个hub与link，也可以为下一层的维度模型的fact表提供参考

#### Hub表

Hub所使用的business key需要注意使用范围，避免冲突。business key多用于join操作，使用int类型比char类型更具性能优势

Hub的4+1属性：
* Hash Key - 用于join操作、主键、外键。hash key应该是可计算的（允许重复生成），固定长度的。建议的计算方式见Chapt.11。hash key不能暴露于Data Vault结构之外
* Business Key - hub的核心内容；对于复合bk，不要求只用一列存储；一个hub对应一个bk，如果一个业务实体有多个bk，那应当用多个hub分别存储
* Load Date - 进入dw系统的时间戳，一旦生成永不可改，用于审计
* Record Source - 也用于数据审计和溯源，如果bk有多个源头，就填写master源头
* Last Seen Date - 可选属性。遇到只能全量加载数据，而不能增量加载数据时，可以标示出一个bk是否已被源头删除，等等。

#### Link表

Link用于关联多个Hub，如果Hub之间的关系从1-1变成了n-n或者1-n，那么Link的存在保证了表关系变更范围降低到了最小：增加新的link表就好了。

Link还可以关联位于不同数据库系统里的Hub。例如一个在Postgres，另一个在MongoDB。

Link的粒度：Link关联的Hub越多，粒度就越小。有点像维度模型里的fact事实表。

如果需要修改Link的粒度，直接修改表结构是不推荐的。因为相应的ETL工具也需要较大的重构，甚至会发生历史数据丢失的情况。推荐的方式是，新建一张Link表，存储新的关系。旧的Link表则封存起来。

Link的属性：
* Hash Key - 需要从link的特征属性计算得出，方便ETL检查重复数据，以及后续查询的join操作
* Load Fate
* Record Source
* Last Seen Date - 可选属性。
* Dependent child key - 可选属性，某些依赖于link属性才能有意义的属性。例如某本里的页码。需要注意，可能符合该条件的属性应该放在Satellite表中。

#### Satellite表

Satellite表用于存储业务实体或关系的描述信息，往往会随时间变化。这些变化也在Sat表的存储范围内。

一个Sat表关联于一个Link或Hub，用其hash key和修改时间共同标示。需要注意的是，一个Link或Hub可能有多个Sat，各类属性按照业务分类或者数据类型的不同放在不同的Sat表中。

Satellite表是DV中唯一能够存储历史数据的位置。重构时需要小心，一定要保留好历史数据。也因此，数据只能一次性写入，永不可改。

不同的属性可能保存在不同的Sat表中，建议两种区别方式：
  1. 数据源。
     - 增加数据源时，可以不改动当前Sat表的结构
     - 维持数据的追溯轨迹
     - 数据导入时，不会出现对目标Sat表的竞争使用（IO层面或数据库层面）；保证了并行化导入的效率
     - 不要求实时数据与批量数据同时导入
  2. 数据变动频率

Sat表包含如下属性：

* Record Source
* Parent Hash Key - 构成自身Primary Key的第一部分
* Load Date - 构成自身Primary Key的第二部分，填写进入DW的时间。
* Load End Date - 当前Sat记录失效的时间
* Hash Diff - 可选属性。所有描述信息的hash值。便于快速比对。根据业务规则，也可以剔除某些列
* Extract Date - 可选属性。数据从Source系统读取的时间

给Link表加Link的时候，考虑一下是否会提升后期重构的复杂度。

过宽的表会降低查询性能，因为页的大小是有限的。一个页能够存储的行越多，性能越好。

### Business Vault layer

这一层介于Data Warehouse vault 和 Information Mart layer之间。在数据上应用了Soft转换规则，例如构建PIT表和Bridge表。

#### Point in Time table

PIT表是为了提高查询效率，在数据库内根据源数据预先计算，因此无法审计。

当查询因为涉及一个Hub/Link与其对应的多个Sat表而效率低下的时候，建议引入PIT表到Business Vault或Information Mart层。PIT表也可以包含计算结果列。

PIT实际上是某一时间的数据快照。当查询对时效性不敏感时，使用PIT可以避免过多的join操作。

为了保证PIT表的查询效率，应该尽量减少表内的数据量，及时清理陈旧的数据。

#### Bridge table

同样是为了减少join次数，提高查询性能，预先计算好放置在Business Vault层中，Bridge表像一种特殊的Link表，其中包含了多个Link/Hub表的Hash key。同时也要有Snapshot date存储行数据的生成时间。

Bridge表中也可以放置一些推导列，但是为出于性能考虑，表列不能太宽。过宽的Bridge表，会抵减这一表结构带来的性能提升。

### Information Delivery/Mart layer

这一层对接终端用户，其中的information是面向主题的，可以用于关系型报告或者OLAP的多维cube计算。

Information Mart Layer通常使用维度模型（在关系型数据库中为星形模型）来建模，也就是1张fact表关联多张dimension表的形式。

Fact表中的dimension数量称作表粒度。fact表粒度小，则有助于终端用户更自由的组织数据，粒度大可以帮助用户做一些预先计算。

这一层可以不存储任何数据，只做虚拟表。也可以出于性能考虑，复制一些数据。因此建议使用RAID0磁盘架构。


### Metric layer

可选层。用于记录运行时数据，运行历史、CPU占用率、内容/磁盘负载，网络吞吐等等。

### Operational Vault layer

可选层。便于操作性应用直接读取或修改Data Warehouse layer的数据，是Data Warehouse layer的拓展。


## 参考资料

* [Data Vault Series. Part 3 - The Business Data Vault](https://www.vertabelo.com/blog/technical-articles/data-vault-series-the-business-data-vault)
