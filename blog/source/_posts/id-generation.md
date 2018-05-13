---
title: 谈谈ID生成方案
date: 2017-08-15 19:11:23
categories: System Design
tags: [id,design]
comments: false
description: "分布式唯一ID是金融系统绕不开的问题"
---
### ID生成一般要符合如下几个原则
- 1 全局唯一
系统设计中对ID的唯一性需求非常普遍。尤其在分布式系统中，要辨别每一次请求是否重复，每一笔业务是否重复，都要求有唯一的ID来识别。对待系统问题查找，审计等有唯一的ID都可以大大简化工作难度和工作强度。
- 2 分布式：
能在本地分布式生成，不用经过网络获取。没有中心单点多点，不依赖于第三方，无限制扩展。这样才能保证ID最大的可靠性和便利性。
- 3 有序递增
ID有序对数据检索极为重要，现代数据库一般都采用B+树索引进行数据检索，有序的ID可以极大提升数据库的效率。递增则可满足排序需求。
- 4 时间相关
ID与时间相关不仅能保证有序递增的，也便于冷热数据的分离。时间相关的ID生成都会面临时间回调问题，要注意处理（保留上次的时间戳，和本次的时间戳比较）
- 5 长度短
ID长度越短其读写效率越高占用空间小，并且ID都会作为索引列，ID越短期生成的索引占用空间也越小。不考虑其他情况的前提下64bit的long型是最佳选择。
- 6 高性能
可能的条件下性能尽可能高，以满足大规模超大规模交易需求。单点每秒百万以上的性能才能称为高性能。
- 7 有内涵
ID的每一部分都有含义，比如包含时间 机器甚至业务信息，便于跟踪定位。尤其在分布式系统下，对线上问题查找原因有极大的帮助。


### 目前有如下一些主流的ID生成方案
#### 1 MySQL自增主键  autoincreasement id
在MySQL中单表自增主键是最高效的数据库索引，通过单库顺序自增或者多库设定不同的offset和increament来保证ID唯一性。但要在数据存入之后再从数据库取回，不便且面临网络可靠性的风险。默认性能不高单库很难突破千/s级别，多库情况下扩展困难。
优缺点：
- [X] 全局唯一
- [] 分布式
- [X] 有序递增（仅单库）
- [] 时间相关
- [X] 长度短
- [] 高性能
- [] 有内涵

#### 2 数据库Sequence
MySQL并不像DB2 oracle有现成的Sequence特性。它是通过数据表+数据库函数方式实现的Sequence。同样也可以通过设定“current_value”和“increment”的方式多库生成。具体方法网上很多但是大部分都是有问题的，并发条件下会有重复ID。这里提供一个完整的方案 [t_sequence.sql](./uploads/posts/id-generation/t_sequence.sql)。默认性能不高也是很难突破千/s级别，多库扩展困难。但是可以使用获得的ID做基础，在本地附加本地顺序号进行二次分配提高性能。如获取1则在本地可分配[1000,1999]以此提高性能，实现会麻烦一些。
优缺点：
- [X] 全局唯一
- [] 分布式
- [X] 有序递增（仅单库）
- [] 时间相关
- [X] 长度短
- [] 高性能
- [] 有内涵


#### 3 Redis
利用Redis单线程模型，获取自增ID。与数据库Sequence类似，也可通过使用不同初始offset和increment来扩展。不过引入了对Redis的依赖。默认但节点性能万/s-十万/s，同样可进行本地二次分配。
优缺点：
- [X] 全局唯一
- [] 分布式
- [X] 有序递增（仅单库）
- [] 时间相关
- [X] 长度短
- [] 高性能
- [] 有内涵

#### 4 UUID
UUID有不同的版本，常见version 1 和 version 4。参见 [RFC4122](http://www.ietf.org/rfc/rfc4122.txt)。UUID长度32字符。    
UUID version 1 是基于时间 机器地址 本地序列号构造。
UUID version 4 是基于安全随机数算法构造的，是jdk默认的UUID实现。
优缺点：
- [X] 全局唯一
- [X] 分布式
- [X] 有序递增（仅version 1，单节点）
- [X] 时间相关（仅version 1）
- [] 长度短
- [X] 高性能
- [] 有内涵（仅version 1）

#### 5 snowflake
64bit的long型分布式ID生成。ID中包含时间戳和自增序列。但是依赖于中心分配DataCenterID 和 Worker ID
优缺点：
- [X] 全局唯一
- [] 分布式
- [X] 有序递增（单节点）
- [X] 时间相关
- [X] 长度短
- [] 高性能
- [X] 有内涵（仅时间戳有意义）

#### 6 ObjectId
MongoDB使用12byte数据结构，16进制编码24个字符。结构上与UUID version 1类似，但是做了一定的改进。
<table border="1">
<caption>ObjectID layout</caption>
<tr>
<td>0</td><td>1</td><td>2</td><td>3</td><td>4</td><td>5</td><td>6</td><td>7</td><td>8</td><td>9</td><td>10</td><td>11</td>
</tr>
<tr>
<td colspan="4">time</td><td colspan="3">machine</td> <td colspan="2">pid</td><td colspan="3">inc</td>
</tr>
</table>
  
虽然由于inc采用随机数初始值大大降低了重复的可能性，但是由于machine采用hash值，理论上还是有可能产生重复ID。可能是考虑到inc随机数初始值的前提下可能性不大，且inc递增使时间回调有一定容忍性（回调n秒时峰值不超过2^24/(n+1)的前提下依能保证唯一），ObjectID也没有考虑时间回调的问题。一切系统放大到一定规模的时候，不太可能出现的问题就一定会出现。
优缺点：
- [] 全局唯一
- [X] 分布式
- [X] 有序递增（单节点）
- [X] 时间相关
- [] 长度短
- [X] 高性能
- [X] 有内涵

### FinTx的分布式唯一ID生成方案
fintx-identifer采用了类似ObjectId的方案，使用15byte数据结构，将machine扩展为48bit使网卡MAC地址作为机器唯一标识保证绝对唯一。处理了时间回调问题。并且采用64进制（Base64）编码字符串，将长度控制在20个字符。由于Base64的字符在UTF-8编码下只占1byte。UTF-8编码的20字符（20byte）唯一ID比UUID的32字符(32byte)减少37.5%。
<table border="1">
<caption>UniqueId layout</caption>
<tr>
<td>0</td><td>1</td><td>2</td><td>3</td><td>4</td><td>5</td><td>6</td><td>7</td><td>8</td><td>9</td><td>10</td><td>11</td><td>12</td><td>13</td><td>14</td>
</tr>
<tr>
<td colspan="4">time</td><td colspan="6">machine</td><td colspan="2">pid</td><td colspan="3">counter</td>
</tr>
</table>

优缺点：    
- [X] 全局唯一（全球唯一）
- [X] 分布式
- [X] 有序递增（单节点）
- [X] 时间相关
- [] 长度短
- [X] 高性能
- [X] 有内涵

欢迎在GitHub上关注FinTx开源的唯一ID生成方案：[fintx-identifer](https://github.com/fintx/fintx-identifer)
