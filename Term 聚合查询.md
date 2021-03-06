# ES Terms聚合  
---
ES Terms聚合是按照某一个字段中的值来进行分类，比如编程语言有java、go、python三门语言，那么就会创建三个桶，分别存放java、go、python的信息。
默认会收集doc_count的信息，记录编程语言为java、go、python的各有多少个。

#### Terms聚合样例：

	GET /_search
	{
	    "aggs" : {
	        "languages" : {
	            "terms" : { "field" : "languages" }
	        }
	    }
	}

#### 得到的结果：
	{
	    "aggregations" : {
	        "languages" : {
	            "doc_count_error_upper_bound": 0, 
	            "sum_other_doc_count": 0, 
	            "buckets" : [ 
	                {
	                    "key" : "java",
	                    "doc_count" : 10
	                },
	                {
	                    "key" : "go",
	                    "doc_count" : 10
	                },
	                {
	                    "key" : "python",
	                    "doc_count" : 10
	                },
	            ]
	        }
	    }
	}

--- 
## 数据不确定性：使用terms聚合，得到的结果可能会带有一定的偏差。  
### 原理：  
比如想要获取错误码code字段中出现频率最高的top5.此时客户端向ES发送聚合请求，主节点接收到请求后，会向每个独立的分片发送请求。分片独立计算
自己分片上的top5错误码，然后返回。当所有的分片结果都返回后，在主节点进行结果的合并，再求出top5并返回给客户端。   

Shard1  |	   Shard2|	    Shard3   
--------|---------------|------------ 
Java   20|	   Go     30|    Python 20  
Go     20|	   PHP    20|	 C      20
Python 10|	   Java   18|	 Java   18  
C#     10|	   C++    16	|    C++    15  
C++    9	 |     C      15|	 PHP    13  
C      8	 |     Python 10|   	 Go     12  
如上表所示，在shard2和shard3上，python和go语言数量没有排到前5，导致计数统计不准确。

--- 
## size与shard_size参数：  

为了改善上述准确性问题，ES提供了size和shard_size两个参数。  
1、size参数规定了最后返回的term个数(默认是10个)  
2、shard_size参数规定了每个分片上返回的个数  
3、如果shard_size小于size，那么分片也会按照size指定的个数计算  
##### 通过这两个参数，如果我们想要返回top5，size=5;shard_size可以设置大于5，这样每个分片返回的词条信息就会增多，相应的误差几率也会减小。
--- 

## 文档计数误差参数：
在进行term聚合时有两个误差参数在结果中显示。  
1、计算文档错误数：doc_count_error_upper_bound，该参数给出没有被计算进最终结果的最大可能数字  

	{
	    "aggregations" : {
	        "products" : {
	            "doc_count_error_upper_bound" : 46,
	            "sum_other_doc_count" : 79,
	            "buckets" : [
	                {
	                    "key" : "Product A",
	                    "doc_count" : 100
	                },
	                {
	                    "key" : "Product Z",
	                    "doc_count" : 52
	                }
	                ...
	            ]
	        }
	    }
	}  

2、每个桶错误数：show_term_doc_count_error，该参数设置为true,会对每个bucket都显示一个错误数，表示最大可能的误差情况，如果不按照排序，这个错误数无法计算出来。  


	GET /_search
	{
	    "aggs" : {
	        "products" : {
	            "terms" : {
	                "field" : "product",
	                "size" : 5,
	                "show_term_doc_count_error": true
	            }
	        }
	    }
	}

---

## order排序参数：
order指定了最后聚合桶结果返回时排序方式，默认按照doc_count进行排序（数量多排在前面）  

	GET /_search
	{
	    "aggs" : {
	        "languages" : {
	            "terms" : { 
	                "field" : "languages",
	                "order" : {"_count":"asc"} 
	            }
	        }
	    }
	}
也可以按照字典方式排序：  

	GET /_search
	{
	    "aggs" : {
	        "languages" : {
	            "terms" : { 
	                "field" : "languages",
	                "order" : {"_term":"asc"} 
	            }
	        }
	    }
	}
还可以指定单值的metric聚合进行排序：  
  
	GET /_search	
	{
	    "aggs" : {
	        "languages" : {
	            "terms" : {
	                "field" : "languages",
	                "order" : { "avg_age" : "desc" }
	            },
	            "aggs" : {
	                "avg_age" : { "avg" : { "field" : "age" } }
	            }
	        }
	    }
	}
同时也支持多值的Metric聚合，不过要指定使用的多值字段：  
  
	GET /_search
	{
	    "aggs" : {
	        "languages" : {
	            "terms" : {
	                "field" : "languages",
	                "order" : { "age_stats.avg" : "desc" }
	            },
	            "aggs" : {
	                "age_stats" : { "stats" : { "field" : "age" } }
	            }
	        }
	    }
	}

--- 
## min_doc_count与shard_min_doc_count参数：
聚合的字段可能存在一些频率很低的词条，如果这些词条数目比例很大，那么就会造成很多不必要的计算。
因此可以通过设置min_doc_count和shard_min_doc_count来规定最小的文档数目，只有满足这个参数要求的个数的词条才会被记录返回。  
1、min_doc_count：规定了最终结果最小文档数的筛选  
2、shard_min_doc_count：规定了分片中计算返回时最小文档数的筛选  
  
	GET /_search
	{
	    "aggs" : {
	        "tags" : {
	            "terms" : {
	                "field" : "tags",
	                "min_doc_count": 10
	            }
	        }
	    }
	}

---
## Filterung过滤值，include和exclude过滤参数：
使用include和exclude可以过滤包含和去除该值的文档:  

	GET /_search
	{
	    "aggs" : {
	        "tags" : {
	            "terms" : {
	                "field" : "tags",
	                "include" : ".*sport.*",
	                "exclude" : "water_.*"
	            }
	        }
	    }
	}
上面的例子中，最后的结果应该包含sport并且不包含water。
也支持数组的方式，定义包含与排除的信息：  

	GET /_search
	{
	    "aggs" : {
	        "JapaneseCars" : {
	             "terms" : {
	                 "field" : "make",
	                 "include" : ["mazda", "honda"]
	             }
	         },
	        "ActiveCarManufacturers" : {
	             "terms" : {
	                 "field" : "make",
	                 "exclude" : ["rover", "jensen"]
	             }
	         }
	    }
	}

--- 
## 多字段聚合：
通常情况，terms聚合都是仅针对于一个字段的聚合。因为该聚合是需要把词条放入一个哈希表中，如果多个字段就会造成n^2的内存消耗。
不过，对于多字段，ES也提供了下面两种方式：  
1、 使用脚本合并字段  
2、 使用copy_to方法，合并两个字段，创建出一个新的字段，对新字段执行单个字段的聚合。

--- 
## 子聚合：
对于子聚合的计算，有两种方式：
1、depth_first 直接进行子聚合的计算    
2、breadth_first 先计算出当前聚合的结果，针对这个结果在对子聚合进行计算。  
默认情况下ES会使用深度优先，不过可以手动设置成广度优先，比如：  

	GET /_search
	{
	    "aggs" : {
	        "actors" : {
	             "terms" : {
	                 "field" : "actors",
	                 "size" : 10,
	                 "collect_mode" : "breadth_first"
	             },
	            "aggs" : {
	                "costars" : {
	                     "terms" : {
	                         "field" : "actors",
	                         "size" : 5
	                     }
	                 }
	            }
	         }
	    }
	}

--- 
## 缺省值Missing Value参数：
缺省值指定了缺省的字段的处理方式：   

	GET /_search
	{
	    "aggs" : {
	        "tags" : {
	             "terms" : {
	                 "field" : "tags",
	                 "missing": "N/A" 
	             }
	         }
	    }
	}