# es_index


## 基本概念  [[basic]]

## 分布式 [[distributed]]




## 倒排索引

ES的倒排索引会针对Json文档中的每个字段建立倒排索引

    可以指定某些字段不做索引

        * 节省存储空间
        * 字段无法被搜索

### 核心组成

* 单词词典（term dictionary）记录所有文档的单词，记录单词倒排列表的关联关系
    
    单词词典一般较大， 可以通过B+树或哈希拉链法实现，满足高性能的插入与查询

* 倒排列表（Posting List）记录了单词对应的文档集合，由倒排索引项组成

    倒排索引项

        文档ID
        词频TF-该单词在文档中出现的次数，用于相关性评分
        位置（Position）- 单词在文档中分词的位置，用于语句搜索（phrase query）
        偏移（offset）- 记录单词的开始结束位置，实现高亮显示





[//begin]: # "Autogenerated link references for markdown compatibility"
[basic]: basic "basic"
[distributed]: distributed "distributed"
[//end]: # "Autogenerated link references"