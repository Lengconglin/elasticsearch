# ES 数据检索
### 数据检索是ES最强大的功能，可以根据关键词、过滤条件等来进行全文检索，找到相关的文档，其步骤如下：  
1、客户端向某个节点发送请求，该节点变为协调节点。  
2、协调节点将请求转发到所有的shard。  
3、query phase:每个shard将自己搜索到的结果(doc id)返回给协调节点，由协调节点进行数据的合并、排序、分页等操作，产生最终的结果。  
4、fetch phase:协调节点根据doc id去各个节点拉取实际的doc数据，并最终返回给客户端。

--- 

### 返回字段参数解释  
	{
	    "took": 0,
	    "timed_out": false,
	    "_shards": {
	        "total": 1,
	        "successful": 1,
	        "skipped": 0,
	        "failed": 0
	    },
	    "hits": {
	        "total": 1,
	        "max_score": 1.0,
	        "hits": [
	            {
	                "_index": "sgy_20200319",
	                "_type": "sgy",
	                "_id": "nt4J83ABM8f5VYbdPsiS",
	                "_score": 1.0,
	                "_source": {
	                    "outInterfaceQueryFlag": "0",
	                    "requestStartDate": 1584625200002,
	                    "requestNo": "no1263829394",
	                    "respSysCode": "system1res",
	                    "titleCode": "title_code",
	                    "retNetworkCode": "000001",
	                    "retReasonCode": "cause02"
	                }
	            }
	        ]
	    }
	}

#### 1、hits  
searchResponse中最重要的部分，它包含了total字段来表示匹配到的文档总数，hits数组默认包含了匹配到的前10条数据，可以通过size来设置返回数量，最大1000。  

  
hits数组中的每个结果都包含_index、_type、_id、_score和_source字段。_source字段包含了存储的业务数据，可以直接使用，这有别于有的搜索引擎只返回文档_id，需要用_id单独取获取文档数据。  

_score字段是相关性得分(relevance score)，它衡量了文档与查询的匹配程度。默认的，返回的结果中关联性最大的文档排在首位，按照_score降序排列。当没有指定任何查询时，所以所有文档的相关性是一样的，因此所有结果的_score都是取得一个中间值1。

max_score指的是所有文档匹配查询中_score的最大值。  

--- 
#### 2、took  
took值代表了整个搜索请求所花费的毫秒数

--- 

#### 3、shards  
_shards节点表明了参与查询的分片数（total字段），有多少是成功的（successful字段），有多少的是失败的（failed字段）。通常我们不希望分片失败，不过这个有可能发生。如果我们遭受一些重大的故障导致主分片和复制分片都故障，那这个分片的数据将无法响应给搜索请求。这种情况下，Elasticsearch会报告分片failed，但仍会继续返回剩余分片上的结果。  

### 4、timeout  
timed_out值指示本次查询是否超时。一般搜索不会出现超时，如果业务需求响应速度比完整的结果更重要，可以设置timeout参数为10ms或者1s。，ES将返回请求超时前收集到的数据。 
  
	GET /_search?timeout=10ms 
超时设置并不是断路器，timeout不会停止执行查询，只是告诉我们目前顺利返回的结果的节点并关闭连接。 尽管已经返回结果了，后台其他分片依旧在执行查询。 

---    
## Search Type  
#### ES提供了两种搜索类型：  
#### 1、query then fetch  
此方式分为两步进行，第一步：先向所有的shard发出请求，各分片只返回排序和排名的相关信息（不包括文档），然后按照各分片返回的分数进行重新排名和排序，取前size个文档。第二步：去相关的shard取文档并高亮需要的字段。这种方式返回的文档可能是用户要求的size的n倍。  

#### 2、DFS query then fetch  
这种方式比第一种方式多了一个初始化散发（initial scatter）步骤，有这一步可以更精确控制搜索打分和排名。这种方式返回的文档数与用户要求的size是相等的。

--- 

## 检索结果处理
由于ES存储数据基本都是打平的业务数据，所以可以将搜索得到的source数据转换成map供业务使用（hit.getSourceAsMap）。

