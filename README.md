# es.application
# es应用场景中的注意点

建议：
	使用es查询时，打印出es查询命令（pretty标准化命令），便于出现bug时利用kibana 辅助进行问题定位

## 1.Es模糊查询的实践

### 文档： 
	英文：https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-prefix-query.html#prefix-query-allow-expensive-queries
	中文：https://doc.codingdict.com/elasticsearch/17/


应用场景下多是前缀模糊查询
https://www.jianshu.com/p/f5d548559791

### 1.text 类型
使用term query，因为es中没有存储整条句子，只存储分词后的单词
->输入整个相同的单词的情况下会搜不到结果
输入句子中的一个单词，大小写不一致
->匹配不到，term是严格查询，es 存储text进行索引时已经全部小写化

使用match query，只要有一个单词和原来句子中的某个单词匹配（不要求大小写一致，索引的时候已经全部转为小写），就会返回该条原来的句子
->可能导致返回过多的不相关的结果

->text类型不合适

### 2.keyword类型
使用term query，严格匹配，无法模糊查询
使用match query，分析器的不同（https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-analyzers.html）
可能产生不同的结果，但是整体不适合该场景的模糊查询


总的来说，es的模糊查询官方是建议慎用的，性能相比其它查询（term，match）而言不太好；
但是如果基于es的存储特性，只使用前缀开始进行模糊查询，则会减少这种性能的开销

### Es模糊查询方法：
https://blog.csdn.net/pony_maggie/article/details/113951893
https://segmentfault.com/a/1190000022100153

前缀搜索的前缀越短结果越多，前缀太短，只有一个字符，那么匹配数据量太多，会影响性能；如果实时性要求较高可以提前进行索引，如使用ngram等

普通前缀索引如果要加速可以引入index_prefixes ，与ngram有一点类似，用更大的索引来换取索引速度
search.allow_expensive_queries 的值设置为 false （默认为 true ）则不支持前缀搜索，但是如果使用了index_prefixes ，提前建了更大的索引则仍然支持前缀搜索（in spite of the search.allow_expensive_queries setting）

前缀语法： 
http://doc.codingdict.com/elasticsearch/263/
https://github.com/olivere/elastic/issues/526
https://cloud.tencent.com/developer/article/1546318


如果long类型数据要前缀收缩则为该字段添加keyword索引
https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-put-mapping.html

## 2.关于es中的分页查询
如果数据量较大，且需要近实时，使用searchafter 缺点是不易于实现向前翻页

## 3.es数据返回条数
### 查询
es查询默认返回最多10000条数据，如果要查出所有数据，注意进行调整，否则会出现数据缺失的情况
如果数据量很大，使用scroll是更好的方法
### 查询并下载
防御式编程，避免搜索条件不当导致下载过大的数据，导致oom
如果确实要下载大量数据，可以通过压缩数据，更改http 请求传输细节等实现（分块传输body），俗称将🐘放进冰箱